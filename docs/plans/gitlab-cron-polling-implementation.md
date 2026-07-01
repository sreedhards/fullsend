# GitLab Cron-Polling Implementation Details

This document contains implementation details for GitLab cron-polling event dispatch in fullsend. For the architectural decision and rationale, see [ADR 0063](../ADRs/0063-gitlab-cron-polling-event-dispatch.md).

## Table of Contents

1. [Dependency Graph](#dependency-graph)
2. [Phase 0: Forge Interface Preparation](#phase-0-forge-interface-preparation)
3. [Phase 1: GitLab Forge Client](#phase-1-gitlab-forge-client)
4. [Phase 2: Cron Poller](#phase-2-cron-poller)
5. [Phase 3: GitLab CI/CD Templates](#phase-3-gitlab-cicd-templates)
6. [Phase 4: CLI Changes](#phase-4-cli-changes)
7. [Phase 5: Integration and Testing](#phase-5-integration-and-testing)
8. [Security-Critical Code Paths](#security-critical-code-paths)
9. [Verification Checklist](#verification-checklist)

## Dependency Graph

```
Phase 0 (forge interface) ──┬──> Phase 1 (GitLab forge client) ──> Phase 4 (CLI changes) ──┐
                            │                                                                │
                            └──> Phase 2 (cron poller) ────────────────────────────────────>├──> Phase 5
                                                                                             │
Phase 3 (CI/CD templates) ─────────────────────────────────────────────────────────────────>─┘
```

Phases 1 and 2 depend on Phase 0 (forge interface changes). Phase 3 (CI/CD templates) has no code dependency on Phase 0 and can start immediately. Phase 4 depends on Phase 1. Phase 5 depends on all prior phases.

## Phase 0: Forge Interface Preparation

**Goal**: Prepare `forge.Client` for multi-forge support without breaking GitHub. Pure refactoring — no behavioral changes.

### New methods on `forge.Client`

Add to `internal/forge/forge.go`:

```go
IsProtectedBranch(ctx context.Context, owner, repo, branch string) (bool, error)
CreatePipelineSchedule(ctx context.Context, owner, repo, ref, description, cron string, variables map[string]string) (scheduleID string, err error)
DeletePipelineSchedule(ctx context.Context, owner, repo, scheduleID string) error
ListPipelineSchedules(ctx context.Context, owner, repo string) ([]PipelineSchedule, error)
UpdateVariable(ctx context.Context, owner, repo, key, value string) error
CreateProtectedVariable(ctx context.Context, owner, repo, key, value string) error
```

These methods are forge-neutral by design. `IsProtectedBranch` maps to GitHub's branch protection API and GitLab's protected branches API. `CreatePipelineSchedule` and `DeletePipelineSchedule` are GitLab-native; the GitHub implementation returns `ErrNotSupported`. `UpdateVariable` maps to GitLab's CI/CD variable API. `CreateProtectedVariable` creates a CI/CD variable with `Protected: true, Masked: false` — used for poll state variables (watermark, label state) that must not be accessible on non-protected branches but whose values are not secrets.

### New sentinel error

```go
var ErrNotSupported = errors.New("operation not supported by this forge")
```

This complements the existing sentinels in `forge.go`: `ErrNotFound`, `ErrAlreadyExists`, and `ErrBranchProtected`.

GitHub returns `ErrNotSupported` for `CreatePipelineSchedule`, `DeletePipelineSchedule`. GitLab returns it for `DispatchWorkflow`, `ListOrgInstallations`, `GetAppClientID`, and org-level secret/variable methods.

**Decision rule**: Use extension interfaces (`GitHubExtensions`) for methods that conceptually do not exist on the other platform (e.g., `ListOrgInstallations`, `GetAppClientID` — GitHub App concepts with no GitLab analogue). Use `ErrNotSupported` for methods with a forge-neutral contract that one forge does not implement yet (e.g., `CreatePipelineSchedule` on GitHub). Callers of extension-interface methods use a type-assertion gate; callers of `ErrNotSupported` methods handle the error per call site.

**Caller handling**: Audit all call sites via `grep -rn 'MethodName' internal/` to build a call-site inventory. Expected handling per call site:
- `DispatchWorkflow` callers (dispatch layer): check for `ErrNotSupported`, fall back to schedule-based dispatch for GitLab
- `DispatchWorkflow` callers (enrollment layer, `internal/layers/enrollment.go` lines 159, 348): repo-maintenance dispatch after enrollment/unenrollment. Skip with a log warning on `ErrNotSupported` — GitLab per-repo installs do not use cross-repo repo-maintenance workflows; enrollment changes are applied directly
- `CreateOrgSecret`/`OrgSecretExists` callers (secrets layer): skip with a log warning when `ErrNotSupported` — per-repo GitLab does not use org-level secrets
- `ListOrgInstallations`/`GetAppClientID` callers (appsetup, CLI): already gated behind `GitHubExtensions` type-assertion, so `ErrNotSupported` is never reached
- `GetLatestWorkflowRun`/`ListWorkflowRuns` callers: skip with a log warning — GitLab uses pipeline status via different mechanisms

### Extension interface

Move GitHub-only methods to a `GitHubExtensions` interface:

```go
type GitHubExtensions interface {
    ListOrgInstallations(ctx context.Context, org string) ([]Installation, error)
    GetAppClientID(ctx context.Context, slug string) (string, error)
}
```

Callers type-assert to access these methods. This keeps the core `forge.Client` interface forge-neutral.

### Forge detection

New file `internal/forge/detect.go`:

```go
func DetectForge(remoteURL string) (string, error) {
    u, err := url.Parse(remoteURL)
    if err != nil {
        return "", fmt.Errorf("invalid remote URL: %w", err)
    }
    host := strings.ToLower(u.Hostname())

    switch host {
    case "github.com":
        return "github", nil
    case "gitlab.com":
        return "gitlab", nil
    default:
        return "", fmt.Errorf("unknown forge host %q: use --forge flag for self-hosted instances", host)
    }
}
```

### Files

| Action | Path |
|--------|------|
| Modify | `internal/forge/forge.go` — add methods, sentinel, extension interface |
| Modify | `internal/forge/github/github.go` — implement new methods (schedule → `ErrNotSupported`; `IsProtectedBranch` → branch protection API); move `ListOrgInstallations`/`GetAppClientID` to `GitHubExtensions` |
| Modify | `internal/forge/fake.go` — implement new methods on FakeClient |
| Modify | `internal/appsetup/appsetup.go` — update `ListOrgInstallations`/`GetAppClientID` calls to use `GitHubExtensions` type-assertion |
| Modify | `internal/cli/admin.go` — update `ListOrgInstallations` calls to use `GitHubExtensions` type-assertion |
| Modify | `internal/cli/github.go` — update `GetAppClientID` calls to use `GitHubExtensions` type-assertion |
| Create | `internal/forge/detect.go` |
| Create | `internal/forge/detect_test.go` |

### Verification

`make go-test && make go-vet` — all existing tests pass unchanged.

## Phase 1: GitLab Forge Client

**Goal**: Implement `internal/forge/gitlab/gitlab.go` with the full `forge.Client` interface.

### Constructor

```go
func New(token string, opts ...Option) (*LiveClient, error)
```

Single-token constructor for the bot project access token. The token is used for all REST and GraphQL API calls. Options include `WithBaseURL(url)` for self-hosted instances (default: `https://gitlab.com`).

### Full method mapping

| `forge.Client` method | GitLab SDK / API | Notes |
|---|---|---|
| `GetRepo` | `Projects.GetProject` | Returns project metadata |
| `GetDefaultBranch` | `Projects.GetProject` → `DefaultBranch` | |
| `GetCommit` | `Commits.GetCommit` | |
| `ListCommits` | `Commits.ListCommits` | |
| `CreateBranch` | `Branches.CreateBranch` | |
| `DeleteBranch` | `Branches.DeleteBranch` | |
| `GetBranch` | `Branches.GetBranch` | |
| `GetFileContent` | `RepositoryFiles.GetFile` | Base64 decode content |
| `ListFiles` | `Repositories.ListTree` | Recursive via `Recursive: true` |
| `CreateOrUpdateFile` | `RepositoryFiles.CreateFile` / `UpdateFile` | Check existence first |
| `CreatePR` | `MergeRequests.CreateMergeRequest` | MR, not PR |
| `GetPR` | `MergeRequests.GetMergeRequest` | |
| `ListPRs` | `MergeRequests.ListProjectMergeRequests` | |
| `UpdatePR` | `MergeRequests.UpdateMergeRequest` | |
| `MergePR` | `MergeRequests.AcceptMergeRequest` | |
| `CreatePRComment` | `Notes.CreateMergeRequestNote` | Notes, not comments |
| `ListPRComments` | `Notes.ListMergeRequestNotes` | |
| `CreatePRReview` | Synthesized from notes + approvals | No native review object |
| `RequestPRReviewers` | `MergeRequestApprovals.SetApprovers` | Approvers, not reviewers |
| `ListPRReviews` | Synthesized from notes + approvals | |
| `GetPRDiff` | `MergeRequests.GetMergeRequestDiff` | |
| `AddLabels` | `MergeRequests.UpdateMergeRequest` or `Issues.UpdateIssue` | Labels in update payload |
| `RemoveLabel` | Same as above | Full label list minus removed |
| `CreateIssue` | `Issues.CreateIssue` | |
| `GetIssue` | `Issues.GetIssue` | |
| `ListIssues` | `Issues.ListProjectIssues` | |
| `UpdateIssue` | `Issues.UpdateIssue` | |
| `CreateIssueComment` | `Notes.CreateIssueNote` | |
| `ListIssueComments` | `Notes.ListIssueNotes` | |
| `CreateRepoSecret` | `ProjectVariables.CreateVariable` | With `Protected: true`, `Masked: true` |
| `DeleteRepoSecret` | `ProjectVariables.RemoveVariable` | |
| `CreateOrUpdateRepoVariable` | `ProjectVariables.CreateVariable` / `UpdateVariable` | |
| `IsProtectedBranch` | `ProtectedBranches.GetProtectedBranch` | 404 → not protected |
| `CreatePipelineSchedule` | `PipelineSchedules.CreatePipelineSchedule` | GitLab-specific |
| `DeletePipelineSchedule` | `PipelineSchedules.DeletePipelineSchedule` | GitLab-specific |
| `ListPipelineSchedules` | `PipelineSchedules.ListProjectPipelineSchedules` | For uninstall cleanup |
| `UpdateVariable` | `ProjectVariables.UpdateVariable` | For poll watermark |
| `CreateProtectedVariable` | `ProjectVariables.CreateVariable` | With `Protected: true`, `Masked: false` — for poll state |
| `DispatchWorkflow` | → `ErrNotSupported` | GitHub-only |
| `ListOrgInstallations` | → `GitHubExtensions` (not on base interface) | GitHub-only |
| `GetAppClientID` | → `GitHubExtensions` (not on base interface) | GitHub-only |
| `CreateOrgSecret` | → `ErrNotSupported` | Per-repo only |
| `OrgSecretExists` | → `ErrNotSupported` | Per-repo only |
| `GetLatestWorkflowRun` | → `ErrNotSupported` | GitHub Actions concept |
| `ListWorkflowRuns` | → `ErrNotSupported` | GitHub Actions concept |
| `CommitFiles` | `Commits.CreateCommit` | Multi-file commit |

### Review synthesis

GitLab has no native "review" object like GitHub's pull request review. Reviews are synthesized from:
- **Notes** with suggestion blocks → "changes requested"
- **Approval status** via `MergeRequestApprovals.GetConfiguration` → "approved"
- **Discussion resolution status** → tracks whether feedback has been addressed

The `CreatePRReview` method posts a note and optionally approves/unapproves the MR.

### Additional polling-support methods

These are internal methods on the client struct (not on `forge.Client`), used by the poller:

```go
func (c *LiveClient) ListIssuesUpdatedSince(ctx context.Context, owner, repo string, since time.Time) ([]Issue, error)
func (c *LiveClient) ListMergeRequestsUpdatedSince(ctx context.Context, owner, repo string, since time.Time) ([]MergeRequest, error)
func (c *LiveClient) ListProjectEvents(ctx context.Context, owner, repo string, targetType string, after time.Time) ([]Event, error)
func (c *LiveClient) ListIssueNotes(ctx context.Context, owner, repo string, issueIID int) ([]Note, error)
func (c *LiveClient) ListMergeRequestNotes(ctx context.Context, owner, repo string, mrIID int) ([]Note, error)
func (c *LiveClient) GetVariable(ctx context.Context, owner, repo, key string) (string, error)
```

### Subgroup path handling

GitLab supports deeply nested namespaces (`org/sub1/sub2/project`). The client must URL-encode the full project path for API calls, or use numeric project IDs. The `GetRepo` method resolves `owner/repo` to a project ID, and subsequent calls use the numeric ID.

### Files

| Action | Path |
|--------|------|
| Create | `internal/forge/gitlab/gitlab.go` (~1500-2000 lines) |
| Create | `internal/forge/gitlab/gitlab_test.go` |

## Phase 2: Cron Poller

**Goal**: Implement the event polling logic that runs inside scheduled GitLab CI/CD pipelines. The poller is a Go package compiled into the `fullsend` binary and invoked via `fullsend poll`. No external infrastructure is required — no Cloud Function, no webhook bridge, no separate deployment.

### Architecture

```
fullsend poll
├── Read FULLSEND_LAST_POLL_AT_{FAST,FULL} from CI variable
├── Query GitLab API for changes since last poll
│   ├── GET /projects/:id/issues?updated_after=T
│   ├── GET /projects/:id/merge_requests?updated_after=T
│   └── GET /projects/:id/events?target_type=note&after=D
├── For each changed item with new notes:
│   └── GET /projects/:id/issues/:iid/notes (or merge_requests/:iid/notes)
├── Apply event routing rules → list of (stage, event) pairs
├── Dispatch each via parent-child pipeline trigger
│   └── Create child pipeline with STAGE, EVENT_PAYLOAD_B64, RESOURCE_KEY
├── Update FULLSEND_LAST_POLL_AT_{FAST,FULL} via API
└── Exit
```

### Package structure

```
internal/poll/
├── poll.go           # Main poll loop
├── poll_test.go      # Unit tests
├── events.go         # Event detection and deduplication
├── events_test.go    # Event detection tests
├── dispatch.go       # Child pipeline triggering
└── state.go          # Watermark state management
```

### CLI command

New subcommand `fullsend poll` added to `internal/cli/`:

```go
func newPollCmd() *cobra.Command {
    return &cobra.Command{
        Use:   "poll",
        Short: "Poll GitLab API for new events and dispatch agent stages",
        RunE: func(cmd *cobra.Command, args []string) error {
            forgeToken := os.Getenv("FULLSEND_FORGE_TOKEN")
            projectPath := os.Getenv("CI_PROJECT_PATH")
            gcpProjectID := os.Getenv("FULLSEND_GCP_PROJECT_ID")

            client, err := gitlab.New(forgeToken, gitlabURL)
            if err != nil {
                return err
            }

            poller := poll.New(client, projectPath, poll.Options{
                SlashCommandsOnly: os.Getenv("FULLSEND_POLL_MODE") == "fast",
            })

            return poller.Run(cmd.Context())
        },
    }
}
```

### Poll loop (`poll.go`)

```go
type Poller struct {
    client      *gitlab.LiveClient
    projectPath string
    owner       string
    repo        string
    opts        Options
    accessCache map[int]int // userID → access level, reset per poll cycle
}

type Options struct {
    SlashCommandsOnly bool // fast-poll mode: only check for /fs-* commands
}

func (p *Poller) Run(ctx context.Context) error {
    p.owner, p.repo = splitOwnerRepo(p.projectPath)
    p.accessCache = make(map[int]int)

    // 1. Read watermark
    lastPollAt, err := p.readWatermark(ctx, p.owner, p.repo)
    if err != nil {
        return fmt.Errorf("read watermark: %w", err)
    }

    // 2. Discover events
    var events []RoutableEvent
    var labelState LabelState // non-nil only for full polls
    if p.opts.SlashCommandsOnly {
        events, err = p.discoverSlashCommands(ctx, p.owner, p.repo, lastPollAt)
    } else {
        events, labelState, err = p.discoverAllEvents(ctx, p.owner, p.repo, lastPollAt)
    }
    if err != nil {
        return fmt.Errorf("discover events: %w", err)
    }

    // 3. Deduplicate
    events = p.deduplicate(events)

    // 4. Route and dispatch.
    // Track maxUpdatedAt for successfully dispatched and unroutable events.
    // Separately track minFailedAt — the earliest UpdatedAt among failed
    // dispatches — so the watermark never advances past unprocessed events.
    dispatched := 0
    var maxUpdatedAt time.Time
    var minFailedAt time.Time
    for _, event := range events {
        stage := p.routeEvent(ctx, event)
        if stage == "" {
            if event.UpdatedAt.After(maxUpdatedAt) {
                maxUpdatedAt = event.UpdatedAt
            }
            continue
        }

        if err := p.dispatch(ctx, p.owner, p.repo, stage, event); err != nil {
            log.Printf("dispatch %s for %s failed: %v", stage, event.Key(), err)
            if minFailedAt.IsZero() || event.UpdatedAt.Before(minFailedAt) {
                minFailedAt = event.UpdatedAt
            }
            continue
        }
        dispatched++
        if event.UpdatedAt.After(maxUpdatedAt) {
            maxUpdatedAt = event.UpdatedAt
        }
    }

    // 5. Update watermark (with 30s overlap for clock skew).
    // Only fall back to time.Now() on a truly empty poll (no events
    // discovered). When events exist but all dispatches failed,
    // maxUpdatedAt stays zero and the watermark is not advanced —
    // those events remain in the next poll's lookback window.
    // In the mixed success/failure case, cap maxUpdatedAt at minFailedAt
    // so the window always includes unprocessed failed events.
    if maxUpdatedAt.IsZero() && len(events) == 0 {
        maxUpdatedAt = time.Now()
    }
    if maxUpdatedAt.IsZero() {
        log.Printf("WARNING: all %d dispatches failed, watermark not advanced", len(events))
        return nil
    }
    if !minFailedAt.IsZero() && minFailedAt.Before(maxUpdatedAt) {
        maxUpdatedAt = minFailedAt
    }
    newWatermark := maxUpdatedAt.Add(-30 * time.Second)
    if err := p.updateWatermark(ctx, p.owner, p.repo, newWatermark); err != nil {
        log.Printf("WARNING: failed to update watermark: %v", err)
    }

    // 6. Persist label state after dispatch (deferred from detectNewLabels
    // so that failed dispatches are not marked "seen").
    if labelState != nil {
        p.persistLabelState(ctx, p.owner, p.repo, labelState)
    }

    log.Printf("poll complete: %d events discovered, %d dispatched", len(events), dispatched)
    return nil
}
```

### Event discovery (`events.go`)

```go
type RoutableEvent struct {
    Type         string    // "issue_label", "issue_note", "mr_note", "mr_event"
    IID          int       // issue or MR IID
    UpdatedAt    time.Time
    Labels       []string  // newly-added labels for issue_label; current labels for issue_note
    NoteBody     string    // comment body (for slash command routing)
    NoteID       int       // note ID (for dedup)
    NoteAuthorID int       // note author user ID (for authorization checks)
    IsBot        bool      // whether the note author is a bot
    MRSource     int       // source project ID (for fork MR protection)
    MRTarget     int       // target project ID (for fork MR protection)
}

func (p *Poller) discoverAllEvents(ctx context.Context, owner, repo string, since time.Time) ([]RoutableEvent, LabelState, error) {
    var events []RoutableEvent

    // 1. Issues updated since last poll
    issues, err := p.client.ListIssuesUpdatedSince(ctx, owner, repo, since)
    if err != nil {
        return nil, nil, fmt.Errorf("list issues: %w", err)
    }

    // Detect newly-added labels (state diff against previous poll).
    // On error, abort — continuing with nil newLabels would silently
    // drop all label-based events while the watermark advances past them.
    // Label state is NOT persisted here — the caller persists after
    // dispatch so that failed dispatches are re-detected next poll.
    newLabels, updatedLabelState, err := p.detectNewLabels(ctx, owner, repo, issues)
    if err != nil {
        return nil, nil, fmt.Errorf("detect new labels: %w", err)
    }

    for _, issue := range issues {
        // Check for label-based triggers — one event per newly-added
        // routable label so that multiple labels in the same poll window
        // each dispatch independently.
        if added, ok := newLabels[issue.IID]; ok {
            for _, label := range added {
                events = append(events, RoutableEvent{
                    Type:      "issue_label",
                    IID:       issue.IID,
                    UpdatedAt: issue.UpdatedAt,
                    Labels:    []string{label},
                })
            }
        }

        // Check for new notes on this issue
        notes, err := p.client.ListIssueNotes(ctx, owner, repo, issue.IID)
        if err != nil {
            log.Printf("list notes for issue %d: %v", issue.IID, err)
            continue
        }
        for _, note := range notes {
            if note.CreatedAt.Before(since) {
                continue // skip old notes (client-side filtering)
            }
            events = append(events, RoutableEvent{
                Type:         "issue_note",
                IID:          issue.IID,
                UpdatedAt:    note.CreatedAt,
                NoteBody:     note.Body,
                NoteID:       note.ID,
                NoteAuthorID: note.Author.ID,
                IsBot:        note.Author.Bot,
                Labels:       issue.Labels,
            })
        }
    }

    // 2. MRs updated since last poll (for MR comment-triggered events only —
    //    MR open/update/merge are handled by native CI, not the poller)
    mrs, err := p.client.ListMergeRequestsUpdatedSince(ctx, owner, repo, since)
    if err != nil {
        return nil, nil, fmt.Errorf("list merge requests: %w", err)
    }

    for _, mr := range mrs {
        notes, err := p.client.ListMergeRequestNotes(ctx, owner, repo, mr.IID)
        if err != nil {
            log.Printf("list notes for MR %d: %v", mr.IID, err)
            continue
        }
        for _, note := range notes {
            if note.CreatedAt.Before(since) {
                continue
            }
            events = append(events, RoutableEvent{
                Type:         "mr_note",
                IID:          mr.IID,
                UpdatedAt:    note.CreatedAt,
                NoteBody:     note.Body,
                NoteID:       note.ID,
                NoteAuthorID: note.Author.ID,
                IsBot:        note.Author.Bot,
                MRSource:     mr.SourceProjectID,
                MRTarget:     mr.TargetProjectID,
            })
        }
    }

    return events, updatedLabelState, nil
}

// isProjectAccessTokenBot detects GitLab project access token bot users.
// GitLab's Events API author object does not include a `bot` field.
// Project access token bots use the username pattern "project_{id}_bot_{hash}".
func isProjectAccessTokenBot(username string) bool {
    return strings.HasPrefix(username, "project_") && strings.Contains(username, "_bot_")
}

func (p *Poller) discoverSlashCommands(ctx context.Context, owner, repo string, since time.Time) ([]RoutableEvent, error) {
    // Fast-poll mode: use the Events API to find new notes only.
    // This avoids querying all issues/MRs — just look for note-type events.
    //
    // GitLab Events API response fields used:
    //   evt.Note.NoteableType → "Issue" | "MergeRequest" (mapped to internal event types)
    //   evt.Note.NoteableIID  → issue/MR IID
    //   evt.Note.Body         → comment text (checked for /fs-* prefix)
    //   evt.Note.ID           → note ID
    //   evt.Author.ID         → author user ID (for authorization check)
    //   evt.Author.Username   → username (for bot detection via pattern match)
    //   evt.CreatedAt         → event timestamp
    projectEvents, err := p.client.ListProjectEvents(ctx, owner, repo, "Note", since)
    if err != nil {
        return nil, fmt.Errorf("list note events: %w", err)
    }

    var events []RoutableEvent
    for _, evt := range projectEvents {
        if evt.CreatedAt.Before(since) {
            continue // client-side filtering (Events API after= is date-only)
        }
        // Only include notes that look like slash commands
        if !strings.HasPrefix(evt.Note.Body, "/fs-") {
            continue
        }
        // Normalize NoteableType to internal event type constants.
        // GitLab returns capitalized values ("Issue", "MergeRequest").
        var eventType string
        switch evt.Note.NoteableType {
        case "Issue":
            eventType = "issue_note"
        case "MergeRequest":
            eventType = "mr_note"
        default:
            continue
        }
        events = append(events, RoutableEvent{
            Type:         eventType,
            IID:          evt.Note.NoteableIID,
            UpdatedAt:    evt.CreatedAt,
            NoteBody:     evt.Note.Body,
            NoteID:       evt.Note.ID,
            NoteAuthorID: evt.Author.ID,
            IsBot:        isProjectAccessTokenBot(evt.Author.Username),
        })
    }

    return events, nil
}
```

### Event routing

```go
func (p *Poller) routeEvent(ctx context.Context, event RoutableEvent) string {
    switch event.Type {
    case "issue_label":
        return p.routeIssueLabel(event)
    case "issue_note":
        return p.routeIssueNote(ctx, event)
    case "mr_note":
        return p.routeMRNote(ctx, event)
    default:
        return ""
    }
}

// routeIssueLabel maps label additions to stages.
// No per-user authorization check: adding labels requires Reporter+
// access in GitLab, which is sufficient authorization for triggering
// agent stages. This is intentionally less restrictive than slash
// commands (which require Developer+) because label management is
// a structured workflow action, not free-form command execution.
func (p *Poller) routeIssueLabel(event RoutableEvent) string {
    for _, label := range event.Labels {
        switch label {
        case "ready-to-code":
            return "code"
        case "ready-for-review":
            return "review"
        }
    }
    return ""
}

func (p *Poller) routeIssueNote(ctx context.Context, event RoutableEvent) string {
    if event.IsBot {
        return "" // skip bot comments to prevent re-triggering
    }

    // Slash commands require Developer-level (30+) access to prevent
    // Guest/Reporter users from triggering agent stages.
    body := strings.TrimSpace(event.NoteBody)
    if strings.HasPrefix(body, "/fs-") {
        if !p.hasWriteAccess(ctx, event.NoteAuthorID) {
            log.Printf("slash command from user %d denied: insufficient permissions", event.NoteAuthorID)
            return ""
        }
    }
    switch {
    case strings.HasPrefix(body, "/fs-triage"):
        return "triage"
    case strings.HasPrefix(body, "/fs-code"):
        return "code"
    case strings.HasPrefix(body, "/fs-review"):
        return "review"
    case strings.HasPrefix(body, "/fs-fix"):
        return "fix"
    case strings.HasPrefix(body, "/fs-retro"):
        return "retro"
    case strings.HasPrefix(body, "/fs-prioritize"):
        return "prioritize"
    default:
        // Non-command comment on issue with needs-info label → triage
        for _, label := range event.Labels {
            if label == "needs-info" {
                return "triage"
            }
        }
        return ""
    }
}

func (p *Poller) routeMRNote(ctx context.Context, event RoutableEvent) string {
    if event.IsBot {
        // Only bot-authored changes-requested markers trigger fix.
        // Non-bot comments with this marker are ignored to prevent
        // Reporter-level users from triggering the fix stage.
        if strings.Contains(event.NoteBody, "<!-- fullsend:changes-requested -->") {
            if p.isForkMR(event) {
                return "" // skip fork MRs
            }
            return "fix"
        }
        return ""
    }

    // Slash commands require Developer-level (30+) access to prevent
    // Guest/Reporter users from triggering agent stages.
    body := strings.TrimSpace(event.NoteBody)
    if strings.HasPrefix(body, "/fs-") {
        if !p.hasWriteAccess(ctx, event.NoteAuthorID) {
            log.Printf("slash command from user %d denied: insufficient permissions", event.NoteAuthorID)
            return ""
        }
    }
    var stage string
    switch {
    case strings.HasPrefix(body, "/fs-triage"):
        stage = "triage"
    case strings.HasPrefix(body, "/fs-code"):
        stage = "code"
    case strings.HasPrefix(body, "/fs-review"):
        stage = "review"
    case strings.HasPrefix(body, "/fs-fix"):
        stage = "fix"
    case strings.HasPrefix(body, "/fs-retro"):
        stage = "retro"
    case strings.HasPrefix(body, "/fs-prioritize"):
        stage = "prioritize"
    default:
        return ""
    }

    // Fork MR protection: deny fix/code on fork MRs (or when fork
    // status is unknown, e.g. fast-poll path where MRSource/MRTarget
    // are not populated).
    if (stage == "fix" || stage == "code") && p.isForkMR(event) {
        return ""
    }
    return stage
}

// isForkMR returns true if the MR is a fork (source != target) OR if
// fork status is unknown (zero-valued fields). Deny-by-default: when
// the fast-poll path omits MRSource/MRTarget, fork-sensitive stages
// (fix, code) are blocked rather than silently allowed.
func (p *Poller) isForkMR(event RoutableEvent) bool {
    if event.MRSource == 0 || event.MRTarget == 0 {
        return true // unknown — deny by default
    }
    return event.MRSource != event.MRTarget
}
```

### Authorization

```go
// hasWriteAccess checks whether a user has Developer-level (30+) access
// to the project. Results are cached per poll cycle.
func (p *Poller) hasWriteAccess(ctx context.Context, userID int) bool {
    if access, ok := p.accessCache[userID]; ok {
        return access >= 30 // Developer = 30, Maintainer = 40, Owner = 50
    }

    // Use /members/all/ to include inherited group members, not just
    // direct project members.
    member, err := p.client.GetProjectMemberAll(ctx, p.owner, p.repo, userID)
    if err != nil {
        log.Printf("check member access for user %d: %v (denying)", userID, err)
        p.accessCache[userID] = 0
        return false
    }
    p.accessCache[userID] = member.AccessLevel
    return member.AccessLevel >= 30
}
```

### Deduplication

```go
func (p *Poller) deduplicate(events []RoutableEvent) []RoutableEvent {
    seen := make(map[string]bool)
    var unique []RoutableEvent

    for _, event := range events {
        key := event.Key()
        if seen[key] {
            continue
        }
        seen[key] = true
        unique = append(unique, event)
    }

    return unique
}

func (e RoutableEvent) Key() string {
    if e.NoteID != 0 {
        return fmt.Sprintf("note-%d", e.NoteID)
    }
    return fmt.Sprintf("%s-%d-%s", e.Type, e.IID, strings.Join(e.Labels, ","))
}
```

### Label state tracking

The poller needs to distinguish "label was just added" from "label was already present". Since polling sees only current state (no `changes` object like webhook payloads provide), label change detection is implemented client-side via state comparison.

**Approach**: Store the set of previously-seen labels per issue in a CI/CD variable (`FULLSEND_LABEL_STATE`), encoded as JSON. On each poll, diff current labels against stored state. Only newly-appearing labels trigger routing.

```go
type LabelState map[int][]string // issue IID → label list

func (p *Poller) detectNewLabels(ctx context.Context, owner, repo string, issues []Issue) (map[int][]string, LabelState, error) {
    // Read stored state
    stateJSON, err := p.client.GetVariable(ctx, owner, repo, "FULLSEND_LABEL_STATE")
    if err != nil {
        if errors.Is(err, forge.ErrNotFound) {
            stateJSON = "{}" // first run — all labels are "new"
        } else {
            return nil, nil, fmt.Errorf("read label state: %w", err)
        }
    }

    var previousState LabelState
    if err := json.Unmarshal([]byte(stateJSON), &previousState); err != nil {
        return nil, nil, fmt.Errorf("unmarshal label state: %w (stored value may exceed GitLab's 10,000-char variable limit)", err)
    }

    newLabels := make(map[int][]string)

    // Merge into previous state rather than replacing — only update entries
    // for issues present in the current poll, retaining entries for issues
    // not in the current result set. This prevents spurious "new label"
    // detections when a previously-tracked issue reappears after being
    // absent from the updated_after window.
    for _, issue := range issues {
        prev := previousState[issue.IID]
        prevSet := toSet(prev)

        for _, label := range issue.Labels {
            if !prevSet[label] {
                newLabels[issue.IID] = append(newLabels[issue.IID], label)
            }
        }

        // Update this issue's entry in the persisted state
        previousState[issue.IID] = issue.Labels
    }

    // Prune closed issues to keep state bounded
    for iid := range previousState {
        if p.isIssueClosed(ctx, owner, repo, iid) {
            delete(previousState, iid)
        }
    }

    // Return newLabels and the updated state WITHOUT persisting.
    // The caller persists label state after the dispatch loop so that
    // failed dispatches are not marked as "seen" — they will be
    // re-detected on the next poll.
    return newLabels, previousState, nil
}

func (p *Poller) persistLabelState(ctx context.Context, owner, repo string, state LabelState) {
    stateBytes, _ := json.Marshal(state)
    if err := p.client.UpdateVariable(ctx, owner, repo, "FULLSEND_LABEL_STATE", string(stateBytes)); err != nil {
        log.Printf("WARNING: failed to persist label state: %v", err)
    }
}
```

**CI/CD variable size limit**: GitLab CI/CD variables have a 10,000-character limit. For projects with many issues, the label state JSON may exceed this. Mitigation: only track issues with fullsend-relevant labels (`fullsend:*`), and prune entries for closed issues on each poll. If the state exceeds the limit, fall back to treating all matching labels as "new" (which may cause duplicate dispatches, handled by `resource_group` concurrency control).

### Watermark state management (`state.go`)

```go
func (p *Poller) readWatermark(ctx context.Context, owner, repo string) (time.Time, error) {
    varName := p.watermarkVarName()
    value, err := p.client.GetVariable(ctx, owner, repo, varName)
    if err != nil {
        if errors.Is(err, forge.ErrNotFound) {
            return time.Now().Add(-1 * time.Hour), nil
        }
        return time.Time{}, fmt.Errorf("read watermark %s: %w", varName, err)
    }
    return time.Parse(time.RFC3339, value)
}

func (p *Poller) watermarkVarName() string {
    if p.opts.SlashCommandsOnly {
        return "FULLSEND_LAST_POLL_AT_FAST"
    }
    return "FULLSEND_LAST_POLL_AT_FULL"
}

func (p *Poller) updateWatermark(ctx context.Context, owner, repo string, t time.Time) error {
    return p.client.UpdateVariable(ctx, owner, repo, p.watermarkVarName(), t.Format(time.RFC3339))
}
```

### Child pipeline dispatch (`dispatch.go`)

The poller dispatches agent stages by generating a child pipeline YAML file. The parent pipeline (poll.yml) uses `trigger: include: artifact:` to start child pipelines from the generated YAML. This keeps everything within GitLab's native pipeline hierarchy without requiring trigger tokens.

```go
type Dispatch struct {
    Stage          string `json:"stage"`
    EventType      string `json:"event_type"`
    EventPayloadB64 string `json:"event_payload_b64"`
    ResourceKey    string `json:"resource_key"`
}

func (p *Poller) dispatch(ctx context.Context, owner, repo, stage string, event RoutableEvent) error {
    // Build minimal event payload
    payload := p.buildEventPayload(event)
    payloadB64 := base64.StdEncoding.EncodeToString(payload)

    dispatch := Dispatch{
        Stage:          stage,
        EventType:      event.Type,
        EventPayloadB64: payloadB64,
        ResourceKey:    fmt.Sprintf("%s-%d", event.Type, event.IID),
    }

    // Append to dispatches file (read by generate-child-pipelines job)
    p.appendDispatch(dispatch)
    return nil
}
```

**Child pipeline YAML generation:**

```go
func (p *Poller) generateChildPipelineYAML(dispatches []Dispatch) string {
    var buf bytes.Buffer
    for i, d := range dispatches {
        fmt.Fprintf(&buf, "agent-%d:\n", i)
        fmt.Fprintf(&buf, "  trigger:\n")
        fmt.Fprintf(&buf, "    include: .gitlab/ci/fullsend-%s.yml\n", d.Stage)
        fmt.Fprintf(&buf, "    strategy: depend\n")
        fmt.Fprintf(&buf, "  variables:\n")
        fmt.Fprintf(&buf, "    STAGE: %q\n", d.Stage)
        fmt.Fprintf(&buf, "    EVENT_TYPE: %q\n", d.EventType)
        fmt.Fprintf(&buf, "    EVENT_PAYLOAD_B64: %q\n", d.EventPayloadB64)
        fmt.Fprintf(&buf, "    RESOURCE_KEY: %q\n", d.ResourceKey)
        fmt.Fprintf(&buf, "  rules:\n")
        fmt.Fprintf(&buf, "    - when: always\n")
    }
    return buf.String()
}
```

### Files

| Action | Path |
|--------|------|
| Create | `internal/poll/poll.go` (~300 lines) |
| Create | `internal/poll/poll_test.go` |
| Create | `internal/poll/events.go` (~250 lines) |
| Create | `internal/poll/events_test.go` |
| Create | `internal/poll/dispatch.go` (~150 lines) |
| Create | `internal/poll/state.go` (~80 lines) |
| Modify | `internal/cli/root.go` — add `poll` subcommand |

## Phase 3: GitLab CI/CD Templates

**Goal**: Create pipeline YAML templates that are committed to enrolled projects during install.

### Directory structure

```
internal/scaffold/fullsend-repo-gitlab/
├── .gitlab-ci.yml
├── .gitlab/
│   └── ci/
│       ├── fullsend-dispatch.yml   ← MR event routing (native CI path)
│       ├── fullsend-poll.yml      ← cron poller (scheduled pipeline)
│       ├── fullsend-triage.yml
│       ├── fullsend-code.yml
│       ├── fullsend-review.yml
│       ├── fullsend-fix.yml
│       ├── fullsend-retro.yml
│       └── fullsend-prioritize.yml
└── .fullsend/
    ├── config.yaml
    └── customized/
        ├── agents/.gitkeep
        ├── harness/.gitkeep
        ├── policies/.gitkeep
        ├── skills/.gitkeep
        └── scripts/.gitkeep
```

### Root pipeline (`.gitlab-ci.yml`)

```yaml
include:
  - local: '.gitlab/ci/fullsend-dispatch.yml'
    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  - local: '.gitlab/ci/fullsend-poll.yml'
    rules:
      - if: $CI_PIPELINE_SOURCE == "schedule"
  - local: '.gitlab/ci/fullsend-triage.yml'
  - local: '.gitlab/ci/fullsend-code.yml'
  - local: '.gitlab/ci/fullsend-review.yml'
  - local: '.gitlab/ci/fullsend-fix.yml'
  - local: '.gitlab/ci/fullsend-retro.yml'
  - local: '.gitlab/ci/fullsend-prioritize.yml'

stages:
  - dispatch
  - poll
  - generate
  - agent

workflow:
  rules:
    # Native MR events (review, retro)
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # Scheduled polling (triage, code, slash commands)
    - if: $CI_PIPELINE_SOURCE == "schedule" && $CI_COMMIT_REF_PROTECTED == "true"
    # Child pipelines dispatched by the poller
    - if: $CI_PIPELINE_SOURCE == "parent_pipeline"
```

### MR dispatch (`.gitlab/ci/fullsend-dispatch.yml`)

Handles native MR events — routes `merge_request_event` pipelines to the appropriate agent stage:

```yaml
# fullsend-stage: dispatch (MR events only)

dispatch:
  stage: dispatch
  image: alpine:latest
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  script:
    - |
      set -euo pipefail

      # CI_DEBUG_TRACE guard
      if [ "${CI_DEBUG_TRACE:-}" = "true" ]; then
        echo "ERROR: CI_DEBUG_TRACE enabled — aborting to protect secrets"
        exit 1
      fi

      # GitLab has no CI_MERGE_REQUEST_EVENT_TYPE predefined variable.
      # Determine the MR action by querying its state via the API.
      MR_STATE=$(curl -sf "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}" \
        -H "JOB-TOKEN: ${CI_JOB_TOKEN}" | jq -r '.state')

      case "${MR_STATE}" in
        merged)
          echo "STAGE=retro" >> dispatch.env
          echo "RESOURCE_KEY=mr-${CI_MERGE_REQUEST_IID}" >> dispatch.env
          ;;
        opened)
          echo "STAGE=review" >> dispatch.env
          echo "RESOURCE_KEY=mr-${CI_MERGE_REQUEST_IID}" >> dispatch.env
          ;;
        *)
          echo "Unhandled MR state: ${MR_STATE}"
          touch dispatch.env
          exit 0
          ;;
      esac
  artifacts:
    reports:
      dotenv: dispatch.env
```

### Cron poller pipeline (`.gitlab/ci/fullsend-poll.yml`)

```yaml
# fullsend-stage: poll

poll-events:
  stage: poll
  image: ghcr.io/fullsend-ai/fullsend:latest
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $CI_COMMIT_REF_PROTECTED == "true"
  id_tokens:
    FULLSEND_ID_TOKEN:
      aud: "fullsend"
  variables:
    FULLSEND_FORGE: "gitlab"
  script:
    - |
      set -euo pipefail

      # CI_DEBUG_TRACE guard — critical in variable mode (sole defense
      # against PAT exposure at job init), defense-in-depth in WIF mode.
      if [ "${CI_DEBUG_TRACE:-}" = "true" ]; then
        echo "ERROR: CI_DEBUG_TRACE enabled — aborting to protect secrets"
        exit 1
      fi

      # Credential retrieval — mode selected at install time
      if [ "${FULLSEND_CREDENTIAL_MODE}" = "wif" ]; then
        # WIF mode: exchange OIDC token for GCP credentials, then
        # retrieve bot PAT from Secret Manager
        gcloud auth login --cred-file=<(cat <<CRED
      {
        "type": "external_account",
        "audience": "//iam.googleapis.com/${FULLSEND_WIF_PROVIDER}",
        "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
        "token_url": "https://sts.googleapis.com/v1/token",
        "credential_source": { "file": "${FULLSEND_ID_TOKEN_FILE}" },
        "service_account_impersonation_url": "https://iam.googleapis.com/v1/projects/-/serviceAccounts/${FULLSEND_SA}:generateAccessToken"
      }
      CRED
        )
        export FULLSEND_FORGE_TOKEN=$(gcloud secrets versions access latest \
          --secret="${FULLSEND_BOT_TOKEN_SECRET}" \
          --project="${FULLSEND_GCP_PROJECT_ID}")
      fi
      # In variable mode, FULLSEND_FORGE_TOKEN is already set from the
      # protected CI/CD variable — no retrieval step needed.

      # Run the poller — outputs dispatches.json
      fullsend poll \
        --forge gitlab \
        --project "${CI_PROJECT_PATH}" \
        --gitlab-url "${FULLSEND_GITLAB_URL:-https://gitlab.com}" \
        --output dispatches.json
  artifacts:
    paths:
      - dispatches.json
    expire_in: 1 hour

# Generate dynamic child pipeline YAML from poll results
generate-child-pipelines:
  stage: generate
  image: ghcr.io/fullsend-ai/fullsend:latest
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $CI_COMMIT_REF_PROTECTED == "true"
  needs:
    - job: poll-events
      artifacts: true
  script:
    - |
      set -euo pipefail

      if [ ! -s dispatches.json ] || [ "$(jq length dispatches.json)" = "0" ]; then
        echo "No events to dispatch"
        # Write a no-op child pipeline
        echo 'no-op: { script: ["echo No events"], rules: [{ when: always }] }' > child-pipeline.yml
        exit 0
      fi

      # Generate child pipeline YAML from dispatches
      fullsend poll generate-child-pipeline \
        --dispatches dispatches.json \
        --output child-pipeline.yml
  artifacts:
    paths:
      - child-pipeline.yml
    expire_in: 1 hour

# Trigger child pipelines for each dispatched event
dispatch-agents:
  stage: agent
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $CI_COMMIT_REF_PROTECTED == "true"
  needs:
    - job: generate-child-pipelines
      artifacts: true
  trigger:
    include:
      - artifact: child-pipeline.yml
        job: generate-child-pipelines
    strategy: depend
```

### Stage pipeline template (`.gitlab/ci/fullsend-code.yml`)

All stages use the same credential retrieval flow (WIF or variable mode). Events arrive via parent pipeline variables (from the poller's child pipeline) or via native MR event dispatch.

```yaml
# fullsend-stage: code

code:
  stage: agent
  image: ghcr.io/fullsend-ai/fullsend:latest
  id_tokens:
    FULLSEND_ID_TOKEN:
      aud: "fullsend"
  variables:
    FULLSEND_FORGE: "gitlab"
  script:
    - |
      set -euo pipefail

      # CI_DEBUG_TRACE guard
      if [[ "${CI_DEBUG_TRACE:-}" == "true" ]]; then
        echo "ERROR: CI_DEBUG_TRACE enabled — aborting to protect secrets"
        exit 1
      fi

      # Credential retrieval — mode selected at install time
      if [ "${FULLSEND_CREDENTIAL_MODE}" = "wif" ]; then
        gcloud auth login --cred-file=<(cat <<CRED
      {
        "type": "external_account",
        "audience": "//iam.googleapis.com/${FULLSEND_WIF_PROVIDER}",
        "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
        "token_url": "https://sts.googleapis.com/v1/token",
        "credential_source": { "file": "${FULLSEND_ID_TOKEN_FILE}" },
        "service_account_impersonation_url": "https://iam.googleapis.com/v1/projects/-/serviceAccounts/${FULLSEND_SA}:generateAccessToken"
      }
      CRED
        )
        export FULLSEND_FORGE_TOKEN=$(gcloud secrets versions access latest \
          --secret="${FULLSEND_BOT_TOKEN_SECRET}" \
          --project="${FULLSEND_GCP_PROJECT_ID}")
      fi
      # In variable mode, FULLSEND_FORGE_TOKEN is already available.

      # Decode event payload
      EVENT_PAYLOAD_FILE=$(mktemp)
      trap 'rm -f "${EVENT_PAYLOAD_FILE}"' EXIT
      echo "${EVENT_PAYLOAD_B64}" | base64 -d > "${EVENT_PAYLOAD_FILE}"

      # Prepare workspace (layered content resolution)
      fullsend workspace prepare \
        --forge gitlab \
        --root .fullsend

      # Run the agent
      fullsend run \
        --stage code \
        --source-project "${CI_PROJECT_PATH}" \
        --event-type "${EVENT_TYPE}" \
        --event-payload-file "${EVENT_PAYLOAD_FILE}" \
        --forge gitlab \
        --fullsend-dir .fullsend
  resource_group: "fullsend-code-${RESOURCE_KEY}"
  rules:
    - if: $STAGE == "code"
```

### Stage-specific notes

**fix**: Adds fork MR protection:
```yaml
    - |
      # Fork MR protection
      SOURCE_PROJECT=$(echo "${EVENT_PAYLOAD_B64}" | base64 -d | jq -r '.mr_source_project_id // empty')
      TARGET_PROJECT=$(echo "${EVENT_PAYLOAD_B64}" | base64 -d | jq -r '.mr_target_project_id // empty')
      if [ -n "${SOURCE_PROJECT}" ] && [ -n "${TARGET_PROJECT}" ] && [ "${SOURCE_PROJECT}" != "${TARGET_PROJECT}" ]; then
        echo "Fork MR detected — skipping fix stage"
        exit 0
      fi
```

**review** (via native MR event): When triggered by `merge_request_event`, `CI_MERGE_REQUEST_IID` and other MR variables are available directly from GitLab — no event payload decoding needed. The stage template detects the source and adapts:
```yaml
    - |
      if [ "${CI_PIPELINE_SOURCE}" = "merge_request_event" ]; then
        # Native MR event — build payload from CI variables
        # Query MR state since GitLab has no CI_MERGE_REQUEST_EVENT_TYPE variable
        MR_STATE=$(curl -sf "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}" \
          -H "JOB-TOKEN: ${CI_JOB_TOKEN}" | jq -r '.state')
        EVENT_PAYLOAD_FILE=$(mktemp)
        trap 'rm -f "${EVENT_PAYLOAD_FILE}"' EXIT
        jq -n \
          --arg iid "${CI_MERGE_REQUEST_IID}" \
          --arg state "${MR_STATE}" \
          --arg source "${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME}" \
          --arg target "${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}" \
          '{iid: ($iid|tonumber), state: $state, source_branch: $source, target_branch: $target}' \
          > "${EVENT_PAYLOAD_FILE}"
        EVENT_TYPE="merge_request"
      else
        # Polled event — decode from base64 variable
        EVENT_PAYLOAD_FILE=$(mktemp)
        trap 'rm -f "${EVENT_PAYLOAD_FILE}"' EXIT
        echo "${EVENT_PAYLOAD_B64}" | base64 -d > "${EVENT_PAYLOAD_FILE}"
      fi
```

### Files

| Action | Path |
|--------|------|
| Create | `internal/scaffold/fullsend-repo-gitlab/` (entire tree) |
| Modify | `internal/scaffold/scaffold.go` — add `GitLabPerRepoScaffold()` function |

## Phase 4: CLI Changes

**Goal**: `fullsend admin install group/project --forge gitlab` works end-to-end.

### New flags

On `fullsend admin install`:
- `--forge {github|gitlab}` — auto-detected from remote URL, overridable
- `--gitlab-url` — GitLab instance URL (default: `https://gitlab.com`)
- `--poll-interval` — cron schedule for polling (default: auto-detect from tier)
- `--skip-schedule-create` — skip pipeline schedule creation (for externally managed schedules)

### Token resolution

```go
func resolveGitLabToken() (string, error) {
    if token := os.Getenv("GL_TOKEN"); token != "" {
        return token, nil
    }
    if token := os.Getenv("GITLAB_TOKEN"); token != "" {
        return token, nil
    }
    out, err := exec.Command("glab", "auth", "token").Output()
    if err == nil {
        token := strings.TrimSpace(string(out))
        if token != "" {
            return token, nil
        }
    }
    return "", fmt.Errorf("no GitLab token found: set GL_TOKEN, GITLAB_TOKEN, or run 'glab auth login'")
}
```

### Per-repo enforcement

`fullsend admin install testgroup --forge gitlab` returns an error: "GitLab installation supports per-repo mode only. Provide a group/project path."

### GitLab per-repo install flow

```go
func runGitLabPerRepoInstall(ctx context.Context, target string, opts installOpts) error {
    // 1. Parse group/project
    owner, repo := splitOwnerRepo(target)

    // 2. Resolve token
    token, err := resolveGitLabToken()

    // 3. Create forge client (admin token for setup operations)
    client, err := gitlab.New(token, opts.gitlabURL)

    // 4. Validate project
    project, err := client.GetRepo(ctx, owner, repo)
    // Check user has Maintainer access
    // Check default branch exists

    // 5. Validate default branch is protected
    protected, err := client.IsProtectedBranch(ctx, owner, repo, project.DefaultBranch)
    if !protected {
        return fmt.Errorf("default branch %q is not protected — protect it before installing fullsend", project.DefaultBranch)
    }

    // 6. Check CI_DEBUG_TRACE is not enabled at project or group level.
    // GET /projects/:id/variables/CI_DEBUG_TRACE — if exists and value == "true", fail.
    // Also check group-level: GET /groups/:id/variables for each ancestor group.
    // In variable mode, the script-level guard cannot prevent PAT exposure
    // because GitLab logs CI/CD variables at job init before any script runs.
    // Document that a Maintainer re-adding CI_DEBUG_TRACE after install (at
    // any level) bypasses the guard in variable mode.

    // 7. Create Project Access Token (Developer, api scope)
    // POST /projects/:id/access_tokens
    botPAT := createProjectAccessToken(ctx, client, owner, repo)

    // 8. Store bot PAT — mode depends on --gcp-project flag
    credentialMode := "variable" // default: no GCP required
    if opts.gcpProject != "" {
        credentialMode = "wif"
        // Store PAT in GCP Secret Manager
        storePATInSecretManager(ctx, opts.gcpProject, owner, repo, botPAT)
    } else {
        // Store PAT as a protected, masked CI/CD variable
        client.CreateRepoSecret(ctx, owner, repo, "FULLSEND_FORGE_TOKEN", botPAT)
        maintainerCount := countMaintainers(ctx, client, owner, repo)
        if maintainerCount > 1 {
            log.Warn("Variable mode selected with %d Maintainers. Any Maintainer can "+
                "enable CI_DEBUG_TRACE after install, exposing the bot PAT in job logs. "+
                "Consider using --gcp-project for WIF mode instead.", maintainerCount)
        }
    }

    // 9. Detect GitLab tier for poll interval configuration
    tier := detectGitLabTier(ctx, client, owner, repo)
    pollInterval := determinePollInterval(tier, opts.pollInterval)
    // Free tier: "0 * * * *" (hourly)
    // Premium+: "*/5 * * * *" (every 5 minutes)

    // 10. Create pipeline schedule(s)
    if !opts.skipScheduleCreate {
        if tier == "premium" || tier == "ultimate" {
            // Fast poll: every 5 minutes, slash commands only
            client.CreatePipelineSchedule(ctx, owner, repo, project.DefaultBranch,
                "fullsend fast poll", "*/5 * * * *",
                map[string]string{"FULLSEND_POLL_MODE": "fast"})
            // Slow poll: every 15 minutes, full event scan
            client.CreatePipelineSchedule(ctx, owner, repo, project.DefaultBranch,
                "fullsend full poll", "*/15 * * * *",
                map[string]string{"FULLSEND_POLL_MODE": "full"})
        } else {
            // Free tier: single hourly poll
            client.CreatePipelineSchedule(ctx, owner, repo, project.DefaultBranch,
                "fullsend poll", "0 * * * *", nil)
        }
    }

    // 11. Commit CI/CD template files
    scaffoldFiles := scaffold.GitLabPerRepoScaffold()
    client.CommitFiles(ctx, owner, repo, project.DefaultBranch,
        "chore: add fullsend CI/CD pipeline", scaffoldFiles)

    // 12. Set protected CI/CD variables.
    // Use CreateProtectedVariable (Protected: true, Masked: false) for
    // configuration identifiers — CreateRepoSecret (Protected + Masked)
    // requires values >= 8 characters (e.g. "wif" would fail) and masks
    // GCP resource names in logs, hindering debugging.
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_CREDENTIAL_MODE", credentialMode)
    if credentialMode == "wif" {
        client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_WIF_PROVIDER", wifProviderResourceName)
        client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_SA", serviceAccountEmail)
        client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_BOT_TOKEN_SECRET", secretManagerSecretName)
        client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_GCP_PROJECT_ID", opts.gcpProject)
    }
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_FORGE", "gitlab")
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_PER_REPO_INSTALL", "true")

    // 13. Initialize poll watermarks (protected — must not be accessible
    // to pipelines on non-protected branches to prevent tampering).
    // Separate watermarks for fast-poll (slash commands only) and
    // full-poll (all events) to prevent fast polls from advancing
    // the watermark past unprocessed label/note events.
    initTime := time.Now().Format(time.RFC3339)
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_LAST_POLL_AT_FAST", initTime)
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_LAST_POLL_AT_FULL", initTime)
    client.CreateProtectedVariable(ctx, owner, repo, "FULLSEND_LABEL_STATE", "{}")

    // 14. Set up inference WIF (if --inference-project provided)

    // 15. Print CI minute warning for shared runners
    if tier == "free" {
        log.Warn("Free tier detected. Polling will consume CI minutes on shared runners. " +
            "Consider using self-hosted runners. See ADR 0063 for details.")
    }
}
```

### Tier detection

```go
func detectGitLabTier(ctx context.Context, client *gitlab.LiveClient, owner, repo string) string {
    // Try to create a test pipeline schedule with 5-min interval.
    // If it fails with "is too frequent", we're on Free tier.
    // This is a heuristic — GitLab doesn't expose the tier via API.
    //
    // Alternative: check if project access tokens are available
    // (Premium+ on gitlab.com, all tiers on self-managed).
    //
    // For self-managed instances, assume Premium capabilities
    // (admins can configure any schedule interval).
}
```

### Uninstall flow

```go
func runGitLabPerRepoUninstall(ctx context.Context, target string, opts uninstallOpts) error {
    owner, repo := splitOwnerRepo(target)

    // 1. Delete pipeline schedules
    schedules, _ := client.ListPipelineSchedules(ctx, owner, repo)
    for _, s := range schedules {
        if strings.HasPrefix(s.Description, "fullsend") {
            client.DeletePipelineSchedule(ctx, owner, repo, s.ID)
        }
    }

    // 2. Revoke project access token
    // 3. Clean up credential storage (mode-dependent)
    //    - WIF mode: delete Secret Manager secret, remove WIF attribute condition
    //    - Variable mode: delete FULLSEND_FORGE_TOKEN CI/CD variable
    // 4. Remove CI/CD template files
    // 5. Remove CI/CD variables (FULLSEND_LAST_POLL_AT_FAST, FULLSEND_LAST_POLL_AT_FULL,
    //    FULLSEND_LABEL_STATE, FULLSEND_CREDENTIAL_MODE, FULLSEND_FORGE,
    //    FULLSEND_PER_REPO_INSTALL)
}
```

### Files

| Action | Path |
|--------|------|
| Modify | `internal/cli/admin.go` — add flags, `runGitLabPerRepoInstall()`, token resolution |
| Modify | `internal/cli/root.go` — add `poll` subcommand |
| Create | `internal/cli/poll.go` — `fullsend poll` command |
| Modify | `internal/config/config.go` — add `Forge` field, validation |

## Phase 5: Integration and Testing

### Integration wiring

- `fullsend run --forge gitlab` constructs a GitLab forge client with bot PAT from `FULLSEND_FORGE_TOKEN`
- `fullsend poll --forge gitlab` runs the polling loop
- Config schema accepts `forge: gitlab` in `config.yaml`
- Forge detection integrated into CLI argument parsing

### Unit tests

| Component | Test focus |
|-----------|-----------|
| GitLab forge client | Mock HTTP responses via `httptest.NewServer`. Cover: MR creation, comment posting, label operations. Review synthesis from notes + approvals. Error handling. Subgroup paths. Polling query methods (`ListIssuesUpdatedSince`, etc.). |
| Poller | Event discovery with mock API responses. Slash command detection. Label state diffing. Event routing. Deduplication. Watermark management. Fast-poll vs full-poll modes. |
| Forge detection | GitHub URL → `"github"`. GitLab URL → `"gitlab"`. SSH remote → error. Self-hosted → error with flag suggestion. `--forge` override. |
| CLI | GitLab argument parsing. Per-repo enforcement for GitLab. Token resolution chain. Poll interval selection by tier. |
| Config | `forge: gitlab` validation. Unknown forge rejection. |

### Integration tests

Mock GitLab API → poller → child pipeline generation:
1. Poller discovers new issue with `ready-to-code` label → dispatches code stage
2. Poller discovers `/fs-triage` comment → dispatches triage stage
3. Poller discovers MR comment with changes-requested marker (same project) → dispatches fix stage
4. Poller discovers MR comment with changes-requested marker (fork MR) → skips fix stage
5. Poller skips bot-authored comments → no dispatch
6. Poller handles empty poll (no events since last watermark) → no dispatch, watermark advances to current time
7. Poller deduplicates events across overlapping windows → single dispatch
8. Label state tracking: newly-added label triggers dispatch, pre-existing label does not
9. Full install flow with mock GitLab API (no real GitLab instance)

### E2E tests

Against GitLab.com:
1. Create a test project
2. Run `fullsend admin install group/project --forge gitlab`
3. Verify pipeline schedule(s) created with correct intervals
4. Create an issue with `/fs-triage` comment
5. Wait for next poll cycle → verify triage pipeline fires and triage agent runs
6. Add `ready-to-code` label to issue
7. Wait for next poll cycle → verify code pipeline fires and code agent creates MR
8. Verify review pipeline fires immediately on MR open (native CI path)
9. Run `fullsend admin uninstall group/project --forge gitlab`
10. Verify cleanup (schedule deleted, project access token revoked, variables deleted)

Self-hosted testing: Docker-based GitLab CE instance for version compatibility testing. Minimum GitLab version: 17.0+ (stable trigger API, CI/CD variable protection, pipeline schedules).

### FakeClient updates

Add implementations to `internal/forge/fake.go` for:
- `IsProtectedBranch` — configurable return value
- `CreatePipelineSchedule` — record call, return fake schedule ID
- `DeletePipelineSchedule` — record call
- `UpdateVariable` — record call

## Security-Critical Code Paths

These paths require extra review attention. A bug here is a security vulnerability, not just a functional failure.

### 1. Pipeline schedule targets protected default branch only

**File**: `internal/cli/admin.go` (install flow), `.gitlab-ci.yml` (workflow rules)

The pipeline schedule MUST target the protected default branch. The `workflow:rules` enforce `$CI_COMMIT_REF_PROTECTED == "true"` for scheduled pipelines.

```yaml
# CORRECT — schedule always targets default branch
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" && $CI_COMMIT_REF_PROTECTED == "true"
```

**Consequence of bug**: Pipeline runs on a non-protected branch. In WIF mode, WIF attribute conditions (requiring `ref_protected == "true"`) provide defense-in-depth — the OIDC token exchange fails. In variable mode, protected variable status prevents exposure on non-protected branches.

### 2. Protected variable creation

**File**: `internal/forge/gitlab/gitlab.go`

When creating CI/CD variables for secrets, the `Protected` flag MUST be `true`. Protected variables are only exposed to pipelines running on protected branches.

**Consequence of bug**: Any pipeline (including on MR branches with attacker-modified `.gitlab-ci.yml`) can see credentials. In WIF mode, this exposes WIF configuration (OIDC token replay within ~5 minute TTL). In variable mode, this directly exposes the bot PAT.

### 3. `CI_DEBUG_TRACE` guard

**Files**: All CI/CD template YAML files, `internal/cli/admin.go`

Every stage pipeline must exit early if debug tracing is detected. This prevents credential leakage through verbose job logs. **In variable mode, this guard is the sole defense** — GitLab logs all CI/CD variables at job initialization, before any script runs. In WIF mode, the guard is defense-in-depth — even if bypassed, the PAT is not in a CI/CD variable and is retrieved after the guard runs.

### 4. Fork MR blocking

**File**: `internal/poll/events.go` (poller routing), `.gitlab/ci/fullsend-fix.yml` (pipeline template)

Fork MR protection in three places:
- `isForkMR` helper denies when `source_project_id != target_project_id`
- `isForkMR` also denies when source/target are unknown (zero-valued) — this covers the fast-poll path where MR details are not fetched, ensuring deny-by-default
- Fix pipeline template checks `source_project_id != target_project_id` (defense-in-depth)

**Consequence of bug**: Fork MR triggers fix/code pipeline that pushes commits to the target project.

### 5. Slash command authorization

**File**: `internal/poll/events.go`

The poller MUST verify that slash command authors have Developer-level (30+) project access before dispatching agent stages. The `hasWriteAccess` method queries the GitLab Members API and caches results per poll cycle. Without this check, Guest/Reporter users could post `/fs-code` or `/fs-fix` commands to trigger agent stages.

**Consequence of bug**: Unauthorized users trigger code generation or fix stages, potentially modifying repository contents.

### 6. Event payload base64 encoding

**File**: `internal/poll/dispatch.go`

Event payloads MUST be base64-encoded before passing as child pipeline variables.

**Consequence of bug**: YAML injection via issue titles or MR descriptions containing YAML metacharacters.

### 7. Bot comment filtering

**File**: `internal/poll/events.go`

The poller MUST skip bot-authored comments to prevent the agent's own replies from re-triggering agent stages. Exception: bot-authored comments containing `<!-- fullsend:changes-requested -->` markers must trigger the fix stage.

**Consequence of bug**: Infinite loop — agent posts a comment, poller detects it as a new event, dispatches the stage again.

### 8. Poll state variable protection

**File**: `internal/poll/state.go`

Both `FULLSEND_LAST_POLL_AT_FAST`, `FULLSEND_LAST_POLL_AT_FULL`, and `FULLSEND_LABEL_STATE` MUST be protected (created as protected variables during install). Tampering with any requires Maintainer access — the same privilege level as modifying the pipeline. Separate watermarks prevent fast polls (slash commands only) from advancing past unprocessed label/note events that the full poll handles.

**Consequence of bug**: For the watermark, an attacker could set it far in the future (skipping events) or far in the past (reprocessing old events). Reprocessing is handled by deduplication and `resource_group` concurrency control. Skipping is the higher risk — but requires Maintainer access, which is already within the insider threat model. For the label state, an attacker could clear it so all existing labels re-fire as "new," causing spurious agent stage dispatches.

## Verification Checklist

- [ ] `make go-test` — all unit tests pass (existing + new)
- [ ] `make go-vet` — no issues
- [ ] `make lint` — passes
- [ ] Poller unit test covers: event discovery, slash command detection, label state diffing, routing, dedup
- [ ] Poller unit test verifies bot comment filtering (both skip and changes-requested exception)
- [ ] Poller unit test verifies fork MR protection in routing
- [ ] GitLab client unit test asserts `Protected: true` on secret variable creation
- [ ] All stage YAML files contain `CI_DEBUG_TRACE` guard
- [ ] Fix stage YAML contains fork MR protection
- [ ] `workflow:rules` require `$CI_COMMIT_REF_PROTECTED == "true"` for scheduled pipelines
- [ ] Poll watermark variable created as protected during install
- [ ] Event payloads base64-encoded before passing to child pipelines
- [ ] Child pipeline YAML generation produces valid GitLab CI syntax
- [ ] `fullsend admin install --dry-run testgroup/testproject --forge gitlab` shows correct plan
- [ ] `fullsend admin install testgroup --forge gitlab` returns per-repo enforcement error
- [ ] E2E: Install on GitLab.com test project → pipeline schedules created → issue events detected → agent pipelines fire → uninstall cleans up

---
title: "63. GitLab cron-polling event dispatch"
status: Accepted
relates_to:
  - agent-infrastructure
  - gitlab-implementation
  - security-threat-model
topics:
  - gitlab
  - forge
  - ci-cd
  - per-repo
  - polling
  - cron
---

# 63. GitLab cron-polling event dispatch

Date: 2026-06-13

## Status

Accepted

<!-- ADRs are point-in-time records, but not fully frozen after acceptance.
     Minor annotations are welcome: cross-references to related ADRs, short
     notes linking to newer decisions, or clarifying remarks. However, do not
     substantially rewrite the Context, Decision, or Consequences sections. If
     the decision itself needs to change, write a new ADR that supersedes this
     one. For evolving design narrative, use docs/architecture.md. -->

## Context

Fullsend needs to detect and react to GitLab events — new issues, merge
requests, comments, and label changes — so that agent stages (triage, code,
review, fix, retro) can be dispatched automatically. On GitHub, native event
triggers (`pull_request_target`, `issues`, `issue_comment`) handle this within
GitHub Actions. GitLab has no equivalent for most event types.

GitLab's CI/CD pipeline trigger sources are: `push`, `merge_request_event`,
`schedule`, `trigger`, `web`, `api`, and `parent_pipeline`. Of these, only
`merge_request_event` maps to an agent-relevant event. Issue creation, comment
posting, and label changes have no native CI pipeline trigger. GitLab supports
per-repo installation mode only (no per-org); the pipeline runs inside the
enrolled project on the protected default branch.

See [ADR 0028](0028-gitlab-support.md) (deprecated) for the original GitLab
support architecture discussion. ADR 0028 documented a webhook bridge approach;
this ADR supersedes that direction based on the operational complexity analysis
in Options 1–3 below. [ADR 0045](0045-forge-portable-harness-schema.md)
defines the forge-portable harness schema that GitLab stage templates must
conform to.

## Options

### Option 1: Webhook bridge Cloud Function

Deploy a GCP Cloud Function that receives GitLab webhook POST requests,
validates the `X-Gitlab-Token` header, and calls the Pipeline Trigger API to
dispatch agent stages.

**Rejected.** Requires external infrastructure (Cloud Function) that must be
deployed, monitored, and secured. Exposes a public HTTPS endpoint — an inbound
attack surface. Requires three credential types per project (bot PAT, webhook
secret, trigger token). Creates a complex deployment story for self-hosted
GitLab behind corporate firewalls (VPN peering, on-premise containers, or
Cloud Run + VPC Connector). The bridge cannot be eliminated even in a hybrid
model — if any event type uses webhooks, the full bridge must be deployed.

### Option 2: Webhook-only (all events via bridge)

Use the webhook bridge for all events, eliminating native CI triggers.

**Rejected.** Still requires the bridge with all its operational complexity.
The correct response to "if we need webhooks for some events, why not all?" is
to eliminate the bridge entirely, not to double down on it.

### Option 3: Native MR events + webhook bridge for issues/comments

Use GitLab's native `merge_request_event` for MR events, keep the webhook
bridge only for issues and comments.

**Rejected.** Still requires the bridge Cloud Function. The bridge's
operational cost is dominated by deployment, monitoring, and credential
management — not by event type count.

### Option 4: Pure cron polling (no native CI triggers)

Poll for all events including MR creation and updates.

**Rejected.** MR events have a viable native CI path (`merge_request_event` +
`include: local:`) with sub-minute latency and zero additional infrastructure.
Polling for MRs adds unnecessary latency to the most frequent, most
latency-sensitive operation (code review).

## Decision

GitLab event dispatch uses a **two-path model**:

1. **Native CI triggers for MR events.** MR creation, update, reopen, and
   merge trigger pipelines via GitLab's `merge_request_event` pipeline source.
   The dispatch template is loaded via `include: local:` from the protected
   default branch, ensuring untrusted MR branches cannot modify dispatch logic.

2. **Cron-polled events for everything else.** A scheduled pipeline runs every
   N minutes (5 minutes on Premium/Ultimate, 60 minutes on Free tier), queries
   the GitLab API for new issues, comments, and label changes since the last
   poll, and dispatches agent stages via parent-child pipelines.

No external infrastructure is required for event dispatch — no webhook bridge,
no webhook secrets, no trigger tokens.

```
ENROLLED PROJECT                           GCP (optional, WIF mode only)
────────────────                           ────
.gitlab-ci.yml (root pipeline)             WIF pool/provider (validates GitLab OIDC)
.gitlab/ci/fullsend-dispatch.yml (MR routing)  Service Account (impersonated by jobs)
.gitlab/ci/fullsend-poll.yml (cron poller)     Secret Manager:
.gitlab/ci/fullsend-triage.yml … retro.yml       - bot PAT per enrolled project
.fullsend/ (config workspace)

MR events (native CI):
  MR opened/updated → merge_request_event → fullsend-dispatch.yml → review/fix stage

Issues, comments, labels (cron):
  Pipeline schedule (5 min) → fullsend-poll.yml → GitLab API → dispatch agent stage

Credentials (WIF mode):
  Pipeline job → OIDC token → GCP STS → WIF → SA → Secret Manager → bot PAT

Credentials (variable mode):
  Pipeline job → protected CI/CD variable FULLSEND_FORGE_TOKEN → bot PAT
```

### Credential model

A Developer-role project access token with `api` scope, created during
`fullsend admin install`. Two storage modes are supported:

**Mode 1: OIDC/WIF (recommended).** The bot PAT is stored in GCP Secret
Manager and retrieved at runtime via GitLab OIDC → GCP WIF. No secrets
are stored as CI/CD variables. This is the recommended mode when GCP
infrastructure is available (e.g., projects already using Vertex AI for
inference).

**Mode 2: Protected CI/CD variable (fallback).** The bot PAT is stored
as a protected, masked CI/CD variable (`FULLSEND_FORGE_TOKEN`). No GCP
infrastructure required. This is the default mode for environments
without GCP access, including self-hosted GitLab instances with no cloud
dependency.

The install flow selects the mode automatically: if `--gcp-project` is
provided, OIDC/WIF is configured; otherwise, the CI/CD variable path is
used. A `FULLSEND_CREDENTIAL_MODE` protected variable (`wif` or
`variable`) tells pipeline templates which retrieval path to execute.

Key properties shared by both modes:

- **Single credential type.** One bot PAT per project handles all REST and
  GraphQL operations. No webhook secrets, trigger tokens, or mint service.
- **Bot identity.** The project access token creates a dedicated bot user,
  providing attributable identity equivalent to GitHub Apps.
- **GraphQL support.** Unlike `CI_JOB_TOKEN`, the bot PAT authenticates
  GraphQL — required for GitLab's Work Items API.

OIDC/WIF mode additionally provides:

- **`CI_DEBUG_TRACE` defense-in-depth.** GitLab logs all CI/CD variables
  at job initialization, *before* any script runs. In variable mode, a
  Maintainer enabling `CI_DEBUG_TRACE` exposes the PAT in job logs before
  the script-level guard can abort. In WIF mode, only WIF configuration
  variables (pool IDs, project numbers) are logged — the PAT is retrieved
  later by `gcloud`, after the guard has already run. This is the primary
  security difference between the two modes.
- **Cryptographic access control.** WIF attribute conditions restrict
  token retrieval to the enrolled project on protected branches
  (`assertion.project_id` + `assertion.ref_protected == "true"`).
- **Separation of administrative domains.** WIF configuration lives in
  GCP IAM, outside the GitLab Maintainer's control. A GitLab Maintainer
  cannot modify WIF attribute conditions without GCP IAM access.
- **No token mint.** Standard GCP WIF replaces the custom mint Cloud
  Function used for GitHub.

### Cron poller

The poller runs as `fullsend poll` inside the fullsend container image,
invoked by a scheduled pipeline on the protected default branch. It:

1. Reads a timestamp watermark (`FULLSEND_LAST_POLL_AT`, protected CI/CD
   variable) marking the last successful poll.
2. Queries the GitLab API for issues, MRs, and notes updated since the
   watermark (with a 30-second overlap for clock skew).
3. Routes detected events to agent stages based on labels, slash commands, and
   state changes.
4. Dispatches stages via dynamically-generated child pipeline YAML.
5. Advances the watermark.

Label change detection uses client-side state diffing — the poller tracks
previously-seen labels per issue and triggers only on newly-added labels. This
compensates for the lack of a `changes` object that webhook payloads provide.

**Multi-frequency polling (Premium/Ultimate):** Two pipeline schedules — a
fast poll (every 5 minutes, slash commands only) and a slow poll (every 15
minutes, full event scan). On Free tier, a single hourly poll is the only
option.

### Event routing

The design goal is **event parity with GitHub** — users see the same labels,
slash commands, and stage dispatches regardless of forge. The table below
documents how each event maps to the two-path transport model (native CI vs
cron polling), not a new event specification.

| Detected Change | Transport | Stage |
|---|---|---|
| Issue label `ready-to-code` added | Cron poll (label state diff) | code |
| Issue label `ready-for-review` added | Cron poll (label state diff) | review |
| Issue note starting with `/fs-{triage,code,review,fix,retro,prioritize}` | Cron poll (note body prefix) | corresponding stage |
| Issue note (non-command) on issue with `needs-info` label | Cron poll (label check) | triage |
| MR opened/updated/reopened | Native CI (`merge_request_event`) | review |
| MR merged | Native CI (`merge_request_event`) | retro |
| MR note with `<!-- fullsend:changes-requested -->` | Cron poll (note body marker) | fix (same-project MRs only) |

Bot-authored comments are skipped to prevent re-triggering loops (exception:
the `changes-requested` marker from the review agent).

### Slash command latency

Slash commands (`/fs-*`) are the only latency-sensitive operation. Mitigations:

- **Labels as primary triggers.** Applying `ready-for-review` or
  `ready-to-code` labels is discoverable, visible, and has no polling latency
  on the native MR CI path.
- **Multi-frequency polling** keeps slash command latency to 5 minutes on
  Premium/Ultimate.
- **Manual pipeline trigger** via the GitLab UI as a power-user escape hatch.

**MR note limitation (fast-poll):** `/fs-fix` and `/fs-code` commands on MR
notes are only acted upon during the full-poll cycle (every 15 minutes on
Premium/Ultimate), not the fast poll. The fast-poll path does not fetch MR
source/target project IDs, so the fork MR protection check (deny-by-default
when unknown) blocks these stages. This adds up to 10 minutes of latency
beyond the fast-poll interval. Fetching MR details per note in fast-poll
would add API calls that defeat its lightweight purpose. In practice, fix
stages are typically triggered by the review bot's `changes-requested`
marker (which uses the full-poll path), not human slash commands.

**Quick Action risk:** GitLab may silently strip unrecognized `/`-prefixed
lines. If confirmed empirically, GitLab should use an alternative prefix
(`fs:triage` or `@fullsend triage`). [ADR 0042](0042-fs-prefix-for-slash-commands.md)
permits forge-specific syntax.

### GitLab tier considerations

| Feature | Free | Premium | Ultimate |
|---|---|---|---|
| Schedule minimum interval | 60 min | 5 min | 5 min |
| Project access tokens (SaaS) | Not available | Available | Available |
| CODEOWNERS enforcement | Not available | Available | Available |
| CI minutes (shared runners) | 400/month | 10,000/month | 50,000/month |

**Free tier** is functional but degraded: 60-minute poll interval, no project
access tokens on gitlab.com (must use personal access token), no CODEOWNERS
guardrails, and CI minute quota is insufficient for polling on shared runners.
Self-hosted runners are required. As an alternative, Free tier users can run
`fullsend poll` on an external scheduler (cron on a VM, Kubernetes CronJob,
etc.) at any desired interval. This reintroduces external infrastructure but
is architecturally simpler than a webhook bridge — the poller is entirely
outbound (no public endpoint, no inbound payload parsing) and uses the same
code path as the in-CI poller.

**Premium** (recommended minimum): 5-minute polling, project access tokens,
CODEOWNERS enforcement, adequate CI minutes for a single project.

`fullsend admin install` adapts poll frequency and interaction model to the
detected tier.

### Security model

The security model follows the project's threat priority order (external
injection > insider > drift > supply chain):

- **No inbound attack surface.** Polling is entirely outbound — no public
  endpoint, no webhook parser, no shared-secret authentication.
- **Protected branch enforcement.** `workflow:rules` require
  `$CI_COMMIT_REF_PROTECTED == "true"` for scheduled pipelines.
- **Protected CI/CD variables.** All fullsend CI/CD variables are marked
  protected — accessible only to pipelines on protected branches.
- **`CI_DEBUG_TRACE` guard.** Install-time validation and runtime abort if
  debug tracing is detected. In **variable mode**, this guard is the sole
  defense against PAT exposure via debug tracing — GitLab logs CI/CD
  variables at job init, before any script runs. In **WIF mode**, the guard
  is defense-in-depth — even if bypassed, the PAT is not in a CI/CD
  variable and is retrieved after the guard runs. **Known limitation:**
  install-time validation checks project-level and group-level variables but
  cannot query instance-level CI/CD variables (requires admin API access).
  On self-hosted GitLab instances where instance admins are outside the
  trusted team, WIF mode is recommended.
- **Event data sanitization.** Attacker-controlled content is base64-encoded
  before passing to child pipelines.
- **Fork MR protection.** Fix/code stages are skipped when
  `source_project_id != target_project_id`.
- **Slash command authorization.** Only users with Developer-level (30+)
  project access can trigger agent stages via `/fs-*` commands.

**Security comparison of credential modes:**

| Threat vector | WIF mode | Variable mode |
|---|---|---|
| `CI_DEBUG_TRACE` by Maintainer | PAT not exposed (defense-in-depth) | PAT exposed at job init (guard is sole defense) |
| Maintainer marks branch as protected | WIF grants token (same risk) | Variable exposed (same risk) |
| GitLab database compromise | PAT not in GitLab (in Secret Manager) | PAT stored in GitLab |
| Admin domain separation | WIF config requires GCP IAM | All within GitLab RBAC |
| Audit trail | GCP Data Access logs | GitLab audit logs (Premium+) |

WIF mode is recommended for projects where the Maintainer pool extends
beyond trusted team members, or where compliance requires external
secret storage.

### Forge abstraction

[ADR 0005](0005-forge-abstraction-layer.md) requires new forges to implement
`forge.Client`. This ADR adds forge-neutral methods:

- `IsProtectedBranch` — maps to GitHub branch protection API and GitLab
  protected branches API
- `CreatePipelineSchedule` / `DeletePipelineSchedule` — GitLab-native; GitHub
  returns `ErrNotSupported`
- `UpdateVariable` — for poll watermark management

A new `ErrNotSupported` sentinel (complementing the existing `ErrNotFound`,
`ErrAlreadyExists`, and `ErrBranchProtected` sentinels) allows forge
implementations to reject inapplicable operations. GitHub-only methods
(`ListOrgInstallations`, `GetAppClientID`) move to a `GitHubExtensions`
extension interface. This requires interface evolution beyond pure
implementation — adding methods to `forge.Client` and refactoring
GitHub-specific methods into an extension interface. This is anticipated
growth of the abstraction boundary, not a violation of
[ADR 0005](0005-forge-abstraction-layer.md)'s design; the changes to
`appsetup.go` and `admin.go` are limited to calling new forge-neutral
methods rather than adding forge-conditional logic.

## Consequences

**What becomes easier:**

- **No external infrastructure for event dispatch.** No Cloud Function, no
  webhook bridge. Self-hosted GitLab requires only outbound HTTPS.
- **Single credential per project.** One bot PAT, stored in either GCP
  Secret Manager (WIF mode) or as a protected CI/CD variable (variable
  mode). No webhook secrets, trigger tokens, or mint service changes.
- **Stronger event authenticity.** Events read directly from the GitLab API,
  not from potentially spoofed webhook payloads.
- **No event loss.** Polling reads from the source of truth. Webhooks can fail
  silently or auto-disable after 4 consecutive failures.
- **Simpler emergency shutdown.** Disable the pipeline schedule or revoke the
  bot PAT. No bridge to tear down.
- **MR review latency is unaffected.** Native `merge_request_event` provides
  sub-second triggering for the highest-frequency operation.
- **Tier-adaptive.** Works on all GitLab tiers with graceful degradation.
- **No GCP requirement.** Variable mode allows deployment on self-hosted
  GitLab with no cloud dependency. WIF mode reuses GCP infrastructure
  already provisioned for Vertex AI inference.

**What becomes harder or changes:**

- **Issue/comment event latency.** Up to 5 minutes on Premium, 60 minutes on
  Free. Acceptable for asynchronous agent operations, poor for interactive use
  on Free tier.
- **CI minute consumption.** Polling runs continuously. At 5-minute intervals:
  ~8,640 min/month on shared runners. Self-hosted runners are not billed.
- **State management.** The poller must track watermarks, deduplicate events
  across overlapping windows, and diff label state. This state is internal
  to the GitLab forge implementation and does not leak into the
  `forge.Client` interface, preserving the forge-neutral contract from
  [ADR 0005](0005-forge-abstraction-layer.md).
- **Slash command latency.** Up to 5 minutes vs sub-second with webhooks.
  Labels mitigate this for common operations.
- **Quick Action stripping.** GitLab may strip `/fs-*` commands from comments.
  Requires testing and potentially alternative syntax.
- **Per-repo only.** No centralized config or credential management across
  projects.
- **`api` scope is broad.** Narrower scopes are not available in GitLab today.

**Risks** (ordered by threat priority):

1. **YAML injection in child pipeline generation.** Attacker-controlled
   issue/MR content could break child pipeline YAML syntax. Mitigated by
   base64 encoding of event payloads passed to child pipelines.
1. **Prompt injection via polled events.** Attacker-controlled issue/MR
   content reaches the agent at inference time. This risk is identical
   across all forges and is handled by the existing agent harness security
   layer, not by the transport mechanism.
2. **Watermark tampering.** A Maintainer could skip or replay events by
   modifying `FULLSEND_LAST_POLL_AT`. Mitigated by protected variable status
   and event deduplication.
3. **Schedule modification.** A Maintainer could retarget the schedule to a
   non-protected branch. In WIF mode, mitigated by WIF attribute conditions
   rejecting credential retrieval. In variable mode, mitigated by protected
   variable status (not exposed on non-protected branches).
4. **Missed events from API quirks.** The Notes API lacks `created_after`; the
   Events API `after` parameter is date-only. Mitigated by 30-second watermark
   overlap and dual-frequency polling as reconciliation.

**Comparison with GitHub:**

| Concern | GitHub | GitLab (this ADR) |
|---|---|---|
| Primary credential | App installation token via mint | Bot PAT (WIF or CI/CD variable) |
| MR/PR event dispatch | `pull_request_target` | `merge_request_event` |
| Issue/comment dispatch | Native events (sub-second) | Cron polling (5 min) |
| External infrastructure | Mint Cloud Function | None for event dispatch |
| Credential types | App key + installation token | Single bot PAT |

Detailed implementation guidance is in the companion document:
[docs/plans/gitlab-cron-polling-implementation.md](../plans/gitlab-cron-polling-implementation.md).

---
title: "125. Hybrid GitLab dispatch (webhook fast-path + poller backstop)"
status: Accepted
relates_to:
  - agent-infrastructure
  - gitlab-implementation
  - security-threat-model
topics:
  - gitlab
  - forge
  - ci-cd
  - dispatch
  - webhook
  - polling
  - drivers
---

# 125. Hybrid GitLab dispatch (webhook fast-path + poller backstop)

Date: 2026-09-26

## Status

Accepted

Supersedes the GitLab **dispatch topology** in
[ADR 0067](0067-gitlab-cron-polling-event-dispatch.md) (native-CI two-path,
later pure cron-polling after #7322). ADR 0067's credential model, poller
internals, HMAC dispatch signing, and forge-interface extensions remain
current — **except** that the dispatched-keys persist path must become
concurrency-safe now that the dispatcher is a second writer (see Shared
dispatch spine below). Of ADR 0067's security
guardrails, the protected/masked CI variable
model, the poller reconciliation backstop, and the in-job dispatch gate
remain current; its "no inbound attack surface / no webhook parser / no
trigger token" and "events read only from the GitLab API, never from
spoofable webhook payloads" properties are superseded/relocated by this
ADR's webhook fast-path — see Trust boundaries and guardrails below.

## Context

[ADR 0067](0067-gitlab-cron-polling-event-dispatch.md) chose cron-polling
for GitLab event dispatch to avoid an external webhook-to-trigger
translation bridge. [ADR 0028](0028-gitlab-support.md) had treated that
bridge as required because GitLab webhook JSON and the pipeline trigger
API were not wire-compatible. After #7322, GitLab dispatch is poll-only:
native `merge_request_event` pipelines run on unprotected MR refs, so
protected CI/CD variables (role tokens, `FULLSEND_DISPATCH_SECRET`) are
empty.

A September 2026 spike on the CEE fleet showed that GitLab's native
"use a webhook" pipeline trigger does not need a translation bridge.
`POST /api/v4/projects/:id/ref/:ref/trigger/pipeline?token=` pins
`:ref` in the URL; that ref overrides the payload ref. Pinning it to
the protected default branch makes the resulting `source=trigger`
pipeline a protected-ref job, so it receives protected and masked
CI/CD variables. `TRIGGER_PAYLOAD` is a file-type CI/CD variable: its
environment value is the path to a temporary file holding the webhook
body, not the JSON body itself. Variable injection is server-side and
ref-based, so it is runner-independent.

Event-driven dispatch is already the primary path; polling is the
stated complement ([ADR 0063](0063-polling-based-work-discovery.md)).
A GitLab webhook driver should feed the same normalize → authorize →
CEL → dispatch spine as `gha-event` and `gitlab-poll`
([ADR 0061](0061-harness-cel-dispatch.md),
[ADR 0098](0098-entity-first-harness-evaluation.md)).

## Options

### Option A: Keep pure cron-polling

Retain ADR 0067 after #7322. No new trigger token or webhook
configuration.

**Rejected.** Poll-interval latency is an artifact of giving up native
events, not a GitLab limitation. The spike restored a native fast-path
without reintroducing a bridge.

### Option B: Restore native `merge_request_event`

Return MR pipelines to GitLab's `merge_request_event` source.

**Rejected.** Those pipelines still run on unprotected
`refs/merge-requests/N/head`, so protected variables stay empty. That
is the #7293 / #7322 failure mode.

### Option C: External webhook-to-trigger bridge

Deploy a Cloud Function or similar inbound receiver that translates
webhook JSON into trigger-API form parameters
([ADR 0028](0028-gitlab-support.md) Open Questions).

**Rejected.** The URL-pinned native trigger needs no intermediary and
no inbound endpoint we operate (GitLab → GitLab). A bridge reintroduces
the operational and attack-surface cost ADR 0067 avoided.

### Option D: Hybrid native webhook + poller backstop

Project webhook → native trigger on the protected default branch for
low-latency dispatch; keep the ADR 0067 cron-poller as the
authoritative reconciliation backstop.

**Accepted.**

## Decision

GitLab event dispatch is a **hybrid**:

1. **Webhook fast-path.** A project webhook fires GitLab's native
   "use a webhook" pipeline trigger with `:ref` pinned to the
   protected default branch. The resulting pipeline is
   `CI_PIPELINE_SOURCE=trigger` on a protected ref, so it receives
   protected and masked variables (role tokens,
   `FULLSEND_DISPATCH_SECRET`).
2. **Poller reconciliation backstop.** The existing `gitlab-poll`
   scheduled pipelines remain the source of truth and self-heal
   missed or auto-disabled deliveries. Native `merge_request_event`
   dispatch stays removed.

No translation bridge. No inbound endpoint we operate.

### Scope and non-goals

This ADR records the **dispatch-topology decision** and its trust
boundaries. It does not specify implementation mechanics: the MUST-level
event-normalization, identity-verification, and payload-handling rules
live in the normative
[NormalizedEvent v1](../normative/normalized-event/v1/) `gitlab-webhook`
adapter row, and the remaining ship-gated build-out (enumerated in
Consequences) will be tracked in follow-on implementation issues —
separate from [#7758](https://github.com/fullsend-ai/fullsend/issues/7758),
the documentation issue this ADR closes.

**Non-goals.** Runner routing and CI fleet capacity are pre-existing
GitLab infrastructure concerns, independent of the dispatch topology; the
poller and dispatcher are ordinary API-only jobs and need no runner
changes beyond what the poller already requires.

### Shared dispatch spine

A `gitlab-webhook` input driver is another driver into the shared CEL
dispatch core, not a parallel path. The webhook starts a lightweight
**dispatcher** pipeline that runs the existing
`normalize → authorize → CEL → CreatePipeline` spine, reusing the router,
dispatch HMAC keying, and in-job security gate.

The dispatcher treats the `$TRIGGER_PAYLOAD` contents as **untrusted
hinting only** (resource identifiers, event object kind, and action). Its
provenance splits by field. The **entity, state, actor, and labels** are
populated **solely** from a project-pinned re-fetch of the referenced
issue, merge request, note, and actor via the GitLab API, failing closed
on any missing or mismatched resource. The **transition**
(`transition.kind`, and `source.raw_type`/`raw_action`) has no
API-snapshot equivalent — a `GET` on a merge request cannot by itself
distinguish opened from labeled from synchronized from merged, and a `GET`
on a note cannot distinguish added from edited — so it is derived from the
payload's object-kind/action hint and then **validated for consistency
against the re-fetched snapshot**, failing closed when the claimed action
contradicts current state (e.g. a `merged` transition against a merge
request whose `merged_at` is unset, or a `label_changed` whose named label
is not present/absent as claimed) or when the action cannot be checked.
This is a **narrower** trust split than `gha-event`, which likewise takes
`transition.kind` from the event name/action but re-fetches only
permission and change-proposal metadata; unlike `gha-event`, the
`gitlab-webhook` payload arrives on a trigger a token holder controls, so
its action claim is untrusted until the snapshot check passes — it is
**not** a pure snapshot re-fetch. Those checks bind the action to *current
state*, not to a unique recent occurrence, so **snapshot-consistent replay**
(a token holder re-firing an action that still matches current state, e.g.
replaying an old note as a new comment) is a residual trigger-token risk
the poller/dedup backstop narrows but does not fully close; it is
ship-gating (see Consequences). The normative
[NormalizedEvent v1](../normative/normalized-event/v1/) `gitlab-webhook`
adapter row must specify this field-split provenance, enumerate the
per-transition consistency checks and the required-sub-object provenance,
and require event-time evidence to bind the action to a recent occurrence
wherever the re-fetch exposes it.

Every trigger-supplied variable is attacker-influenced, so the re-fetch
must pin the target-project identity to a source the caller cannot
redirect — not to an overridable predefined variable such as
`CI_PROJECT_ID` or the API-URL variables. `CI_JOB_TOKEN` is that source:
GitLab scopes it to the running job and a same-named trigger variable
cannot override it, so it is the **sole project-identity source**. An
`id_tokens` JWT is **not** an independent identity source — it lands in an
ordinary job environment variable that a same-named trigger variable can
outrank, so "trust the runner-provided token" is not by itself an
enforceable control; if a JWT is used at all it must be bound to the
`CI_JOB_TOKEN`-authenticated job record rather than trusted standing
alone. `CreatePipeline` continues to use the poller-role PAT so it
satisfies the existing in-job identity gate, and any resource whose
project does not match the pinned identity fails closed. The API **base
URL** has no equivalent unforgeable pin yet (`CI_SERVER_URL` is in the
same overridable class); until one exists, base-URL trust is a residual
risk that must be closed before the fast-path ships (see Consequences).
The MUST-level identity mechanics — the `CI_JOB_TOKEN` job-record lookup,
and the claim set and JWKS verification any bound JWT must pass — are
specified in the normative
[NormalizedEvent v1](../normative/normalized-event/v1/) `gitlab-webhook`
adapter row.

Deduplicating a webhook-dispatched event against a later poll-detected one
is **new work required by this ADR**, not existing behavior. The webhook
dispatcher and the cron-poller become concurrent writers on the poller's
per-mode dispatch state, which was designed around a single writer and
persists today with a last-writer-wins write that has no compare-and-swap.
The decision: both writers must dedup through a **concurrency-safe,
conflict-detecting persist** — read the current per-mode state, union the
new key, and commit under an optimistic-concurrency check that fails and
retries if another writer advanced the state since the read — so neither
silently overwrites the other's keys, and each event deduplicates on the
same per-mode state the corresponding poller mode already uses (slash
commands versus all other events). Making the persist path conflict-detecting
is in scope for this ADR, because the current path cannot detect a concurrent
write; the concrete concurrency primitive and the key/branch encoding are
implementation concerns for the normative spec and the follow-on tracking
issue. Until that contract exists, the poller re-dispatches every
webhook-handled event.

This matches [ADR 0098](0098-entity-first-harness-evaluation.md) /
[ADR 0106](0106-serialize-agent-runs-and-coalesce-subsequent-events.md):
the webhook is a low-latency **candidate**; each run reconciles current
entity state; the poller is the scheduled backstop.

### CI scaffold changes

A `trigger`-sourced pipeline does not run under the current GitLab CI
scaffold. Admitting it must **not** be done by extending the agent job's
rules or the in-job `.source` allow-list: that would let a trigger-token
holder supply `STAGE` and invoke the privileged agent script without the
dispatcher's HMAC.

Instead the scaffold gains a **dedicated dispatcher include and job**,
separate from the agent job. The `trigger` source is admitted only for the
dispatcher — at both the root `workflow:rules` block and the dispatcher
include's rules — and never for the agent job. The dispatcher's only path
to an agent job is the same `CreatePipeline` (`.source == "api"`) call the
poller already uses, so the fast-path adds no new dispatcher→agent bypass;
the agent gate itself, however, must be hardened against the new
trigger-token principal this ADR introduces (see below).

The pipeline-source comparisons in those rules are a scheduling filter, not
the isolation boundary: `CI_PIPELINE_SOURCE` is caller-forgeable, which is
why the in-job gate re-derives the source from the Pipelines API and
verifies the dispatch HMAC. But this ADR gives a trigger-token holder the
ability to start a pipeline **directly on the protected ref**, so the agent
gate must resist the same variable-outranking attacks the dispatcher-identity
pin already resists — otherwise those two checks do **not** by themselves
stop the new principal from running the privileged agent body. Two hardening
requirements follow, both **new work** for the agent job and not only the
dispatcher: (1) the in-job re-derivation must take the job, pipeline, and
project identity it checks from the `CI_JOB_TOKEN` job record — the same
sole pin used for dispatcher identity — not from the trigger-overridable
`CI_PIPELINE_ID`, `CI_PROJECT_ID`, or API-URL variables, so a caller cannot
retarget the `.source`/creator check at a legitimate `api` pipeline; and
(2) `FULLSEND_DISPATCH_SECRET` (and the role tokens) must not be outrankable
by a same-named trigger or pipeline variable — the same outrank analysis
already applied to `id_tokens` JWTs — enforced by a GitLab-side control that
restricts user-defined pipeline variables so only the GitLab-injected
`TRIGGER_PAYLOAD` survives, and/or a fail-closed verifier that rejects a
secret whose value could have been caller-supplied. The **poller job** needs
the same identity pin for the same reason: it admits today on a forgeable
`CI_PIPELINE_SOURCE == schedule` and uses its PAT with no in-job `.source`
check, so the new trigger-token principal can reach its schedule-only
admission on the protected ref; the poller must therefore also take its
pipeline source and project identity from the `CI_JOB_TOKEN` job record and
fail closed unless that server-side source is `schedule`. The three jobs'
in-job source allowlists are **disjoint, not a union**: the poller admits
only `schedule`, only the dispatcher admits the webhook's server-side source
(`trigger`), and the agent job admits only `api`. This matters because
GitLab has no distinct "dispatcher" pipeline source — the dispatcher's
server-side source is `trigger`, the same value any trigger-token caller
produces — so a poller allowlist that also accepted `trigger` would let a
forger satisfy the poller's `schedule` YAML rule and then have the in-job
`CI_JOB_TOKEN` pin observe the real `trigger` source and admit the job
anyway, handing the new principal the poller's PAT with no HMAC gate. The
specific YAML encoding of these controls is a
follow-on tracking-issue concern.

### Trust boundaries and guardrails

Two channels must be authenticated independently:

- **GitLab → dispatcher (webhook).** The `$TRIGGER_PAYLOAD` contents are
  untrusted; the trigger token attests only that *a* caller holds it, never
  the payload's contents. Authenticity comes from the dispatcher's
  project-pinned API re-fetch above — the relocation, to the dispatcher, of
  ADR 0067's "events read from the API, not from spoofable payloads"
  property (only partially preserved until the base-URL gap closes).
- **Dispatcher → agent.** The HMAC over dispatch variables and the in-job
  `.source == "api"` gate authenticate this channel only; they do not
  attest the webhook's origin.
- **No event loss.** Webhook delivery can fail or auto-disable; the poller
  remains the reconciliation backstop.

#### Trigger-token threat model

The trigger token starts the privileged dispatcher on the protected ref,
which then receives role tokens and `FULLSEND_DISPATCH_SECRET`. Unlike
masked variables, a trigger URL/token is typically visible to project
Maintainers and in delivery logs, and the trigger API accepts arbitrary
caller-supplied variables. Therefore:

- Treat possession of the token as equivalent to the ability to run the
  privileged dispatcher; define rotation and revocation procedures (who may
  mint it, where the webhook URL is stored, how to revoke and reissue on
  leak).
- Never log the webhook URL or token. A script-level abort is not a
  secret-exposure control here: GitLab materializes protected CI/CD
  variables into the job environment at init and, with debug tracing
  enabled, prints them before any script runs — and a trigger-token holder
  can enable debug tracing through a pipeline variable, widening the
  attacker set past the Maintainer/project-variable scope ADR 0067's
  install-time scan covers. **All** fullsend-managed jobs — dispatcher,
  poller, **and the agent job** — must therefore refuse to **start** under
  debug tracing rather than abort mid-script. The agent job needs this as
  much as the dispatcher: its pipeline-source admission rule is a forgeable
  scheduling filter a trigger-token holder can satisfy, and the in-job
  re-check runs too late to prevent secret materialization. Because GitLab
  evaluates `rules` first-match, the refuse-to-start guard must take
  **precedence over the admitting rules** — a deny-before-admit ordering, or
  an equivalent AND-NOT of the debug-trace condition on every admit rule, in
  both `workflow:rules` and any job-level rules — so a pipeline that already
  matched an admit rule cannot skip the guard. A guard merely appended after
  the admit rules would never be reached and would silently leave this window
  open while still appearing ADR-compliant. The specific variables, truthiness
  matching, and YAML encoding of that guard are implementation concerns for the
  tracking issue.
- Treat the `$TRIGGER_PAYLOAD` file as untrusted input. The MUST-level
  payload-handling rules — accept only GitLab's file-type variable (never a
  caller-supplied path override), validate any identifiers read from its
  contents before use in an API path, and never log its contents — live in
  the normative `gitlab-webhook` adapter row.
- Gate dispatcher jobs on a protected ref.
- Derive every security-relevant field from the API re-fetch and treat
  every trigger-supplied variable as untrusted by default for those
  fields — not only an enumerated list. Security-relevant fields include
  at minimum the API base URL, the target-project identity, credentials
  and dispatch secrets, `STAGE`, and the **transport** the re-fetch's HTTP
  client uses: a caller who cannot redirect the base URL itself can still
  redirect or intercept the PAT-bearing traffic through a caller-supplied
  proxy or CA if the client inherits proxy or trust-store settings from the
  trigger-populated environment. This is not a dispatcher-only concern:
  **all** fullsend-managed jobs — dispatcher, poller, **and the agent job** —
  must use an HTTP client that ignores trigger-influenced proxy and CA
  settings and sources its trusted CA only from a runner-provisioned
  location, because the poller uses its PAT with no HMAC gate and the agent
  job's in-job re-check is a token-bearing call that precedes HMAC
  verification. The target-project identity is pinned via `CI_JOB_TOKEN` as
  specified above (with any `id_tokens` JWT bound to that same job record,
  never trusted standing alone); the base-URL pin remains the open residual
  risk.

## Consequences

- GitLab moves toward GitHub-like event latency while keeping
  polling as the complement ([ADR 0063](0063-polling-based-work-discovery.md)).
- No hosted webhook receiver and no translation bridge; GitLab
  delivers to GitLab.
- Protected-variable access is restored for event-driven GitLab
  dispatch without reviving `merge_request_event` on unprotected MR
  refs.
- New work is required before the fast-path is safe to enable: a
  project-pinned, fail-closed API re-fetch of the payload's referenced
  entities (with `CreatePipeline` still issued via the poller-role PAT) that
  splits provenance — entity/state/actor/labels from the snapshot,
  `transition.kind` from the payload action validated for action-vs-state
  consistency against that snapshot; a concurrency-safe, conflict-detecting
  per-mode dedup persist applied to **both** the poller and the dispatcher,
  replacing the current last-writer-wins write; a dedicated dispatcher CI
  include/job that admits the `trigger` source without touching the agent
  job's gate; **agent- and poller-gate hardening** so the in-job `.source`/HMAC checks
  pin their job/pipeline/project identity to the `CI_JOB_TOKEN` job record
  (the poller included, since its schedule-only admission becomes reachable
  by the trigger-token principal) and `FULLSEND_DISPATCH_SECRET`/role tokens
  cannot be outranked by caller-supplied pipeline variables; a refuse-to-start
  guard for debug tracing (deny-before-admit) covering the agent job as well
  as the dispatcher and poller; transport hardening (proxy/CA settings not
  honored from the trigger-populated environment) across every
  fullsend-managed job's HTTP client; payload-path controls; and trigger-token
  rotation/redaction. Two residual risks remain **open and ship-gating**: the
  API base-URL pin, and **snapshot-consistent replay** of a payload action
  that still matches current state (narrowed by the poller/dedup backstop and
  by binding actions to event-time evidence where the re-fetch exposes it, but
  not fully closed). Both must be resolved before the fast-path ships.
- A trigger token and project webhook become install-time
  configuration; the poller, HMAC secret, and in-job gate stay
  required.

## References

- [ADR 0028 — GitLab Support Architecture](0028-gitlab-support.md)
- [ADR 0061 — Harness CEL triggers and fullsend dispatch drivers](0061-harness-cel-dispatch.md)
- [ADR 0063 — Polling-based work discovery via dispatch drivers](0063-polling-based-work-discovery.md)
- [ADR 0067 — GitLab cron-polling event dispatch](0067-gitlab-cron-polling-event-dispatch.md)
- [ADR 0098 — Evaluate harnesses against entities with optional event context](0098-entity-first-harness-evaluation.md)
- [ADR 0106 — Serialize agent runs and coalesce subsequent events](0106-serialize-agent-runs-and-coalesce-subsequent-events.md)
- [NormalizedEvent v1](../normative/normalized-event/v1/)

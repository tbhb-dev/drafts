<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests

| Status | Owner | Updated | Open decisions | Needs from the operator |
| --- | --- | --- | --- | --- |
| Draft | Platform engineering | 2026-11-14 | 0 | Nothing |

Draft of 2026-11-14. It proposes `github-metrics`, a service that tbhb-dev doesn't run today. Platform engineering would build each component it describes, and appendix A labels each factual claim as existing, new, or unverified.

## Summary

Today tbhb-dev can't see how long a pull request (PR) waits for review or merge. It also has no view of one PR's lifecycle as a timeline.

This draft proposes `github-metrics`, a small containerized Go service. A new read-only GitHub App feeds it webhooks, and a scheduled reconciliation run heals what the webhooks miss. It publishes OpenTelemetry (OTel) data only: one trace per PR lifecycle and three timing histograms per repository, sent to the host's local collector.

For the operator, it means one new GitHub App with two read permissions and one public webhook path behind a Cloudflare Tunnel. The operator approved both on 2026-11-14, as D1 and D2 in [decisions.md](decisions.md) record. The first deploy still waits on a threat-model follow-up design, plan step 6.

## Proposal

Platform engineering runs every component that follows.

- **The GitHub App.** It belongs to the tbhb-dev organization and installs on all repositories. Its permissions are read access to pull requests and to metadata, and it subscribes to two PR events.
- **The service.** One container runs one static Go binary with three loops. A webhook receiver takes deliveries, a worker turns PR state into telemetry, and a reconciler polls the GitHub REST API on a schedule.
- **The state store.** A `bbolt` file on a small mounted volume keeps the delivery queue, the delivery dedupe table, the IDs of exported spans, the recorded metric milestones, the per-repository watermarks, and the entity tags (`ETag` values) for conditional requests. Everything else in the container is read-only.
- **The tunnel.** A Cloudflare Tunnel connector on the same host routes only `POST https://github-metrics.tbhb.dev/webhook` to the service. It makes outbound connections only, and the host doesn't forward any port.
- **The export path.** The service sends OTel Protocol (OTLP) data over HTTP with protobuf encoding to the host's local collector, which forwards traces and metrics to the existing backends.

One deployment serves production, and no staging deployment exists. The first release emits traces and metrics but doesn't export logs. Its own logs go to standard output as structured JSON.

## How it works

This design has no diagrams, because this repository has no D2 rendering rules today. The text and appendix C describe the flow instead.

### One path for webhooks and reconciliation

1. A webhook arrives. The receiver checks the method, the media type, and the size, then verifies `X-Hub-Signature-256` before it parses any JSON.
2. Only after the signature passes, one `bbolt` transaction checks the dedupe table, records the delivery, and appends the raw body to the queue. The receiver syncs the file before it answers `202 Accepted`, so a crash after that answer loses nothing.
3. Next, the worker reads the repository and PR number from the signed body, marks the PR for reconciliation, and deletes the raw body in the same transaction. A body that lists `repositories_added` schedules a backfill of those repositories instead.
4. To reconcile the PR, the worker fetches its current state with conditional requests: the PR itself, its reviews, commits, and timeline, which supplies review request times. It compares that state with the record of what it already exported.
5. Finally, the worker exports the spans that just became complete and writes their IDs to the store after the export succeeds, never before. Metric milestones follow the commit-then-record rule in appendix C, and the periodic reader exports them later.

A delivery only triggers a reconcile, and telemetry content always comes from REST responses, never from a webhook body. The scheduled reconciler feeds PRs into step 4 as well, so both sources run the same code. A duplicate, out-of-order, replayed, or missed delivery all end at the same reconcile of the PR's current state on GitHub.

### Trace model

- **IDs.** The trace ID is the first 16 bytes of a digest from the Secure Hash Algorithm (SHA), in its SHA-256 form, over `github-metrics/v1/trace/` plus the PR's GraphQL node ID. Each span ID is the first 8 bytes of a SHA-256 digest over `github-metrics/v1/span/`, the node ID, the span kind, and a key unique within the PR, such as a review's node ID or a commit hash. Webhook and reconciliation runs derive the same IDs, and both write to one trace.
- **Collisions.** For a 128-bit trace ID, the chance of any collision among a billion PRs is near 2^-69. Within one trace of a thousand spans, 64-bit span IDs collide with odds near 2^-45, and the service accepts both risks without checking. OTel treats an all-zero ID as invalid, and in that case the derivation hashes its own output again.
- **Timestamps.** Each span takes its start and end times from GitHub's own timestamps, never from the time the service saw the event.
- **Tree.** The root span `github.pull_request` runs from creation to the first merge or close. Its children are one span per commit, review request, review, reopen, and close, plus one `merge` span, and appendix C lists the times and attributes of each.
- **Events.** Short facts from the REST timeline without their own duration become span events on the root: ready for review, converted to draft, and a force-push to the head branch. Review thread resolution isn't recorded, because REST doesn't expose it, as appendix C explains.

The store lists every exported span ID, and the worker skips any span already in it.

### Emission timing

A PR can remain open for weeks, and OTel can't export a span before it ends. The service exports each child span once its end time exists on GitHub, with the deterministic root span ID as its parent. It exports the root span exactly once, at the PR's first merge or close, with the root events up to then.

Until then, tracing backends show an open PR's children under a missing root span. This service's collector pipeline must not tail-sample, because tail sampling waits only seconds for a trace to complete. Trace retention in the backend must outlast the longest PR, or a PR's early spans age out before its root arrives.

A reopened PR keeps its trace and its one root, which the service never exports again. The reopen, any later close or merge, and any root event after the root's export become child spans of that root. As a result, the root's end time and state describe the first terminal state, and the latest state is on the last `merge` or `close` child.

### Metrics

| Name | Type and unit | Recorded at |
| --- | --- | --- |
| `github.pull_request.time_to_first_review` | Histogram, seconds | The first review by someone other than the author |
| `github.pull_request.time_to_approval` | Histogram, seconds | The first approving review |
| `github.pull_request.time_to_merge` | Histogram, seconds | The merge |

Every duration starts at the PR's creation. Bucket boundaries, in seconds, are 60, 300, 900, 1800, 3600, 7200, 14400, 28800, 86400, 172800, 345600, 604800, 1209600, and 2592000, from one minute to thirty days.

Metric attributes are `vcs.repository.name`, and `github.author.type` with the values `user` or `bot`. Neither the PR number nor the author's login ever becomes a metric attribute, because both grow without bound.

With about 50 repositories, three histograms produce at most 5,100 series. An SDK cardinality limit of 2,000 attribute sets per instrument caps any surprise.

Metrics come from the same reconciled state as the traces. Each milestone counts once, keyed by the PR's node ID and the milestone name, and only when GitHub's timestamp is less than 24 hours old and later than the store's creation time. The worker commits the milestone to the store first and records the observation second, so a crash between the two loses one observation and never counts one twice.

Backfilled history gets traces and no metrics.

### Reconciliation

Every 15 minutes, the reconciler lists each repository's PRs by update time, newest first, with `If-None-Match`. GitHub doesn't count a `304 Not Modified` response against the primary rate limit. For each changed PR, the reconciler walks back to the repository's watermark and reconciles the PR, and once a night it also reconciles every open PR.

An installation of the App gets at least 5,000 requests an hour. One shared request budget of 2,500 an hour covers webhook-triggered reconciles, incremental runs, the nightly sweep, and the backfill, in that priority order. Steady state uses about 450 requests an hour in the worst case, pagination included, about 9% of the 5,000 floor.

The service uses REST only, as requested, and appendix B compares it with GraphQL. Appendix D gives the full budget, the request pacing, the backfill that runs on first install, and the handling of installation tokens and secondary rate limits.

### Export to the collector

The service sends OTLP over HTTP with protobuf to the address in `OTEL_EXPORTER_OTLP_ENDPOINT`, which the deploy sets to the collector's port 4318 on the host. HTTP suffices at this volume and keeps the gRPC dependency tree out of the binary. Because this hop never leaves the host, it uses plain HTTP.

Traces skip the SDK's batch processor: the worker collects the finished spans of one reconcile and exports them in one synchronous call, and it writes a span's ID to the store only after the collector accepts it. If the collector is down, the failed export leaves the store unchanged. The PR then waits in the queue for the next pass.

Metrics use a periodic reader every 60 seconds with cumulative aggregation, so the SDK keeps running totals in memory, not a queue. While the process lives, a collector outage loses nothing, because the next accepted export carries the totals. A crash or restart loses every observation since the last accepted export, which during a collector outage spans the whole outage, and backends read the restart as a counter reset.

Resource attributes are `service.name=github-metrics`, `service.version`, and `deployment.environment.name=production`. Each span also has `vcs.provider.name=github` and `vcs.owner.name=tbhb-dev`.

### Failure handling

- **Before the answer.** The receiver answers 2xx only after signature verification and a synced enqueue. A full disk or a store error answers `503`, and GitHub records the delivery as failed.
- **After the answer.** A delivery that the worker can't parse goes to a dead-letter bucket after five tries, with its delivery ID in the log. When a reconcile fails on a GitHub error or a collector outage, the PR keeps its mark and the worker retries it with backoff until it succeeds.
- **Redelivery.** GitHub doesn't redeliver failed deliveries on its own, and the first release doesn't call its redelivery API, which needs an App-level JSON Web Token (JWT). Reconciliation is the only healer, and it can heal everything because no telemetry depends on a webhook-only fact.
- **Startup.** The service opens the store and resumes the queue. It reports ready after one reconciliation finishes.
- **Outages.** GitHub's deliveries fail during a service outage, and the next reconciliation after restart catches up from the same REST responses.
- **Store loss.** A lost store re-exports every span once and doesn't double metrics, and restoring an older snapshot is forbidden. Appendix C gives the worst cases.

### Webhook security

- The receiver accepts only `POST` with `Content-Type: application/json` on `/webhook`, and answers `405` or `415` to anything else.
- It caps the body at 5 MB to bound memory for unauthenticated requests. GitHub caps payloads at 25 MB, and how the receiver treats a valid delivery above 5 MB is a gate of the follow-up in plan step 6.
- It computes a Hash-based Message Authentication Code (HMAC) with SHA-256 over the raw body. It compares the result with `X-Hub-Signature-256` in constant time before parsing, and during a secret rotation it accepts the current or the previous secret.
- Each log line records the delivery ID, the event name, and the result, never a body, a header value, a token, or a secret.
- The App private key and the webhook secret come from 1Password through the deploy host's secret manager, mounted as read-only files. The image and the environment never contain them.

GitHub's signature covers only the body, so `X-GitHub-Delivery` and `X-GitHub-Event` can change under a valid signature. The dedupe rules that follow, in the same transaction as the enqueue, account for that.

- **Only after authentication.** A request that fails the signature never reads or writes the dedupe table.
- **Body hash first.** If the SHA-256 of the body is already in the table, the receiver drops the delivery and answers `202`, whatever its delivery ID says.
- **Delivery ID second.** If the delivery ID is already in the table with a different body hash, the receiver rejects the request with `400` and logs the ID and both hashes.
- **Bound.** Entries last 7 days, and the table holds at most 200,000 entries, evicting the oldest first.
- **Older replays.** A signed body older than the table's window can arrive again, and it only triggers a reconcile or an idempotent backfill. That's safe while the span and milestone state is intact, and after a store loss it costs no more than the backfill that the loss already triggers.
- **Event header.** The worker reads the repository, the PR, and any `repositories_added` list from the signed body, and uses `X-GitHub-Event` only in log lines.

The tunnel routes only the `/webhook` path for the hostname, and the edge answers `404` to every other path. Health checks use a second listener that the tunnel never routes.

### Data the service keeps and sends

The App's Pull requests read permission reaches more than this service uses. A PR payload carries its title and body, a review payload carries the review's text, and the REST API under the same permission returns PR diffs and the PR files endpoint, so the App can read any code a PR changes (existing, appendix A). The App doesn't have the Contents permission, so it can't read a repository's full contents.

| Data | Where | Kept for |
| --- | --- | --- |
| Raw webhook bodies, with titles and review text | The queue in the store | Until the worker extracts the repository and PR, normally seconds |
| Raw bodies that fail to parse | The dead-letter bucket | 7 days, then deleted |
| Delivery IDs, body hashes, and times | The dedupe table | 7 days, at most 200,000 entries |
| Span IDs, milestones, watermarks, and `ETag` values | The store | The life of the store |

The store file is proposed at mode `0600`, owned by the image's non-root user, on a volume no other service mounts. Final modes, ownership, and whether backups exist at all are gates of the follow-up in plan step 6, and any backup would hold whatever bodies sit in the queue or dead letters at that moment.

Telemetry carries no titles, bodies, comment text, or diffs. It does carry, for private repositories too, repository names, base branch names, PR numbers, logins, and team slugs, and the collector forwards them to the existing backends. Who can read those backends and how long they keep this data is an input the operator owns, to be confirmed by the threat-model follow-up.

### Relationship to the organization collector

A separate, earlier design plans an organization collector container that polls the GitHub API for organization-level signals, such as Actions minutes, workflow run counts, and queue times. Nobody working on this draft has read that design. Its description here is unverified until that design's owner confirms the boundary.

Under the proposed boundary, `github-metrics` covers the PR lifecycle: PRs, reviews, review requests, and commits on PRs. The organization collector keeps continuous integration (CI), Actions, and organization-level counters. Each uses its own App or token, so neither spends the other's rate-limit budget, and neither emits the other's metrics.

### Deployment shape

- **Image.** A distroless static non-root base pinned by digest, with one Go binary built with cgo off.
- **Runtime.** Run on the production container host, with a non-root user in the image, a read-only root filesystem, one writable volume at `/var/lib/github-metrics`, and no added capabilities.
- **Hardening.** The container doesn't use the host network, and the deploy references the image by digest. The service gets no host account of its own and shares the production container host. These are design choices, using only hardening that costs nothing.
- **Delivery.** The service repository publishes the digest-pinned image and a Compose file. The host operator deploys it, and the service repository has no deploy workflow.
- **Config.** Environment variables set the App ID, the installation ID, the listen addresses, the schedule, and the OTLP endpoint, plus file paths for the two secrets.
- **Health.** `/healthz` reports liveness, and `/readyz` reports an open store and one finished reconciliation.
- **Shutdown.** On `SIGTERM`, within 20 seconds, the receiver stops accepting requests and the worker finishes its current PR. Then the meter provider flushes and the store closes.
- **Logs.** Go's `slog` with the JSON handler, on standard output.

Later versions may add log export and other event types, such as issues, discussions, and Actions. The first release leaves them out.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The backend rejects or hides spans dated weeks in the past | Old PR traces don't appear in search | The first export spike sends a backdated trace and searches for it | Unverified |
| Backend trace retention is shorter than a PR's life | Early child spans age out before the root arrives | Set retention past the longest expected PR, and report orphans | Unverified |
| The OTel Go SDK can't set chosen span IDs | The trace model needs another export path | A custom ID generator in the first spike, with OTLP data built directly as the fallback | Unverified |
| `ETag` values change when the installation token rotates | Fewer free `304` answers | The worst-case budget already assumes no `304` | New, budget in appendix D |
| The organization collector already emits PR metrics | Duplicate series and doubled API use | Plan step 4 confirms the boundary with that design's owner before step 5 starts | Unverified |
| An outage longer than 24 hours | Milestones from the outage get traces but no metric observations | Accepted, since the traces still record each duration | New |
| A crash between a span export and its store write | That reconcile's spans export twice | Accepted, since the window is one write long | New |
| A crash during a collector outage | Every metric observation since the last accepted export is lost | Accepted, since milestones are committed first and never double | New |
| The store is lost, or an older snapshot is restored | Every span re-exports; a restore also double-counts recent milestones | Store creation time gates metrics, and restores are forbidden | New, appendix C |
| Unsigned traffic floods the public webhook path | CPU spent on rejected requests | Size cap before reading, HMAC before parsing, and an optional edge rate limit | New |
| Receiver gaps: rejection of missing or malformed signatures and headers, read, write, and header timeouts, connection and concurrency limits, unknown event names, and oversized valid deliveries | Resource exhaustion or silent drops | Gates of the threat-model follow-up, plan step 6 | New, deferred to step 6 |
| Tunnel acceptance tests don't cover the security gates | A route wider than `POST /webhook` goes unnoticed | Gates of the threat-model follow-up, plan step 6 | New, deferred to step 6 |
| Secret files and the store have loose modes or ownership, or the store is corrupt or backed up carelessly | A local reader gets the key or queued bodies, or the service loses its record | Gates of the threat-model follow-up, plan step 6 | New, deferred to step 6 |
| The previous webhook secret is never retired, or the App key never rotates | An old credential keeps working | Gates of the threat-model follow-up, plan step 6 | New, deferred to step 6 |
| Backend access and retention for private-repository names and logins | Wider exposure than the operator intends | The operator confirms it in the threat-model follow-up | Unverified |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The operator answers D1 and D2 | Done: `decisions.md` lists both under Decided, 2026-11-14 | Operator |
| 2 | A spike exports one backdated, deterministic trace to the collector | The trace appears in the backend under its derived ID, with GitHub's timestamps | Platform engineering |
| 3 | The service repository with the receiver, store, worker, and tests | Tests pass for a replayed delivery, a duplicate, a reused delivery ID with a different body, an out-of-order pair, and a bad signature | Platform engineering |
| 4 | The organization collector's owner confirms the boundary | That design or this one records the confirmation | Platform engineering |
| 5 | The reconciler, the shared request budget, and the backfill | A test against recorded responses shows a full run under its request budget and pacing | Platform engineering |
| 6 | A threat-model follow-up design covers the receiver gaps, the tunnel acceptance gates, secret and store file modes, store backup and corruption, secret retirement and key rotation, and backend access and retention | That design merges with each gate specified, and the operator confirms the backend input | Platform engineering |
| 7 | The App from its manifest, the secrets in 1Password, the published image and Compose file, and the host operator's deployment with the tunnel | Step 6 has merged, a test delivery from the App's settings page returns `202`, and a `GET` on any path returns `404` | Platform engineering and the host operator |
| 8 | A dashboard of the three histograms | The dashboard shows each repository's percentiles from live data | Platform engineering |

The first deploy, step 7, waits on step 6.

## Appendices

- [Decisions](decisions.md): none open, then the decided record.
- [A. Evidence](A-evidence.md): the sources behind each claim, labeled existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): the options not taken, and why.
- [C. Trace and metric specification](C-trace-and-metrics.md): the span tree, its IDs, attributes, events, the recording rule, and store loss.
- [D. GitHub App and reconciliation](D-app-and-reconciliation.md): the events and permissions, then the shared request budget.

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: trace and metric specification

Back to [the design](README.md). This appendix specifies the planned trace of one pull request (PR) and the planned metrics, for a service that doesn't exist yet. Every rule here is new, proposed by this design.

## ID derivation

Let `N` be the PR's GraphQL node ID and `H(x)` the Secure Hash Algorithm (SHA) digest SHA-256 of the UTF-8 bytes of `x`.

- **Trace ID:** the first 16 bytes of `H("github-metrics/v1/trace/" + N)`.
- **Span ID:** the first 8 bytes of `H("github-metrics/v1/span/" + N + "/" + kind + "/" + key)`, with the kind and key from the table that follows.
- **Zero IDs:** OpenTelemetry (OTel) rejects an all-zero ID. If a truncated digest is all zero, the service hashes the full digest again and truncates that.
- **Version:** the `v1` prefix lets a later scheme move every new span to fresh IDs without touching old traces.

Collision odds follow the birthday bound, about `n^2 / 2^(b+1)` for `n` IDs of `b` bits. A billion trace IDs give about 2^-69, and a thousand span IDs in one trace give about 2^-45. The service accepts both and doesn't check for collisions.

## Span tree

| Span name | Kind and key | Start | End | Emitted at |
| --- | --- | --- | --- | --- |
| `github.pull_request` | `root`, empty key | PR `created_at` | `merged_at` or `closed_at` of the first terminal state | First merge or close, once |
| `github.pull_request.commit` | `commit`, commit hash | Author date | Committer date | When the commit appears on the PR |
| `github.pull_request.review_request` | `review_request`, with the timeline event's ID as key | The timeline event's `created_at` | The reviewer's next review, or the request's removal | At that end |
| `github.pull_request.review` | `review`, review node ID | Review `submitted_at` | Review `submitted_at` | When GitHub reports the review as submitted |
| `github.pull_request.merge` | `merge`, empty key | Last approving review, or PR creation without one | `merged_at` | At merge |
| `github.pull_request.close` | `close`, the `closed` timeline event's ID | The latest of the last review, the last reopen, and PR creation | `closed_at` | At each close without merge |
| `github.pull_request.reopen` | `reopen`, the `reopened` timeline event's ID | The event's `created_at` | The same time | When the reopen appears on the timeline |
| `github.pull_request.event` | `event`, the timeline event's ID | The event's `created_at` | The same time | For a root event that happens after the service exported the root |

Each child's parent is the root span ID, which every run derives from `N` alone. A review span has zero duration on purpose: the wait before a review belongs to its review request span. A commit's span runs from authoring to committing, and for most commits that's zero or a few seconds.

## Reopen and the exported root

Each PR's root span goes out exactly once, at the first merge, or at the first close. Because nobody has checked how each backend treats a duplicate span (unverified), the service never sends the same span ID twice.

After a reopen, the PR keeps its trace and its one root. Its reopen becomes a `github.pull_request.reopen` child, a later close gets its own `close` child keyed by its timeline event, and a later merge gets the `merge` child. Any root event after the root's export becomes a zero-duration `github.pull_request.event` child with the attribute `github.event.name`.

In a reopened PR, the root's end time and `vcs.change.state` describe the first terminal state rather than the latest one, and readers find the latest state on the last `merge` or `close` child.

## Root span events

Root events take GitHub's timestamps, and every one of them comes from the REST timeline. When the service builds the root at the first terminal state, it adds them to the root.

- `ready_for_review` and `convert_to_draft`, from the timeline events of the same names.
- `head_ref_force_pushed`, from the timeline.

Review thread resolution isn't recorded. Neither the REST timeline nor the review comment endpoints expose a thread's resolved state (checked against the GitHub Docs pages cited in appendix A). A webhook payload has no GitHub timestamp for the change (unverified), and telemetry content comes only from REST responses, never from a webhook body. For that reason, the App doesn't subscribe to `pull_request_review_thread`.

Review comments don't get a span or an attribute, because the service doesn't read comment text and a submitted review already marks the moment.

## Span attributes

| Attribute | Spans | Example |
| --- | --- | --- |
| `vcs.provider.name` | All | `github` |
| `vcs.owner.name` | All | `tbhb-dev` |
| `vcs.repository.name` | All | `wiki` |
| `vcs.change.id` | All | `52` |
| `vcs.change.state` | Root | `merged`, `closed`, or `open` |
| `vcs.ref.base.name` | Root | `main` |
| `github.pull_request.author` | Root | The author's login |
| `github.author.type` | Root | `user` or `bot` |
| `github.review.state` | Review | `approved`, `changes_requested`, or `commented` |
| `github.review.reviewer` | Review and review request | A login or team slug |
| `vcs.ref.head.revision` | Commit | The commit hash |
| `github.event.name` | Event | `ready_for_review` |

The `vcs.*` names follow the OTel semantic conventions for version control, which are still in development, so their exact names are unverified. Names under `github.*` are this service's own. Titles, bodies, comment text, and diffs never appear, but repository names, branch names, logins, and team slugs from private repositories do, and README section "Data the service keeps and sends" covers who sees them.

## Metrics

| Instrument | Unit | Start | End |
| --- | --- | --- | --- |
| `github.pull_request.time_to_first_review` | `s` | PR `created_at` | `submitted_at` of the first review whose author isn't the PR author |
| `github.pull_request.time_to_approval` | `s` | PR `created_at` | `submitted_at` of the first `APPROVED` review |
| `github.pull_request.time_to_merge` | `s` | PR `created_at` | `merged_at` |

Every instrument here is a histogram with explicit bucket boundaries, in seconds, of 60, 300, 900, 1800, 3600, 7200, 14400, 28800, 86400, 172800, 345600, 604800, 1209600, and 2592000. The 14 boundaries make 15 buckets. With the sum and the count, each attribute set exports 17 series.

### Cardinality budget

| Attribute | Values | Assumption |
| --- | --- | --- |
| `vcs.repository.name` | About 50 | Tens of repositories today, with headroom |
| `github.author.type` | 2 | `user` or `bot` |

That gives 50 × 2 × 3 instruments × 17 series, or 5,100 series at most. Each instrument sets an SDK cardinality limit of 2,000 attribute sets, and sets past it fold into the SDK's overflow series.

The author's login stays out because public repositories accept PRs from anyone, which makes the set unbounded. Per-person timing also invites a use of the data nobody asked for. The PR number never becomes a metric attribute.

### Recording rule

1. After each reconcile, the worker checks the three milestones of the PR.
2. A milestone is eligible if GitHub's state shows it and the store has no record of it, provided its end timestamp is both less than 24 hours old and later than the store's creation time.
3. The worker commits `(N, milestone)` to the store in its own synced transaction.
4. Only after that commit succeeds does the worker record the duration on the histogram. If the commit fails, the worker doesn't record anything, and the next reconcile tries again.
5. A milestone that isn't eligible on age or store creation time gets a store record and no observation, so backfills and long outages never produce a spike of old values.

The worker commits first and records second. A crash between steps 3 and 4 permanently loses that one observation, because the store already marks the milestone as counted. The design accepts a missed count over a double count.

### Metric loss on a crash

Metrics leave through the SDK's periodic reader every 60 seconds with cumulative aggregation, which keeps running totals in memory instead of a queue of observations. A crash or restart loses every observation recorded since the last export the collector accepted. That's normally at most 60 seconds. During a collector outage, though, the loss covers the whole outage, with no time bound.

Memory use doesn't grow during an outage, because cumulative state keeps one total per attribute set, capped at 5,100 series. Lost observations stay lost, since the worker already committed their milestones and won't record them a second time. Backends see a counter reset.

### Store loss and restore

The store is the record of what the service already exported, so losing it or rolling it back makes the service export again.

- **Lost or corrupt store.** The service starts with a new empty store and records the store's creation time. The backfill then exports every span of every PR in scope a second time with the same IDs, and the backend's handling of those duplicates is unverified. Metrics don't double, because step 2 skips any milestone that ended before the store's creation time.
- **Restored older snapshot.** The operator must not restore an older copy of the store, because the snapshot keeps its old creation time. If someone restores one anyway, every span exported after the snapshot exports again, and every milestone counted after the snapshot that's still under 24 hours old counts twice.
- **Instead of a restore,** the operator deletes the damaged file and lets the service start empty. The threat-model follow-up that README plan step 6 names sets the store's file modes and corruption handling, and it decides whether anyone backs up the store at all.

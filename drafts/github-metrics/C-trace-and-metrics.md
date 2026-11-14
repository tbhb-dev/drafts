<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: trace and metric specification

Back to [the design](README.md). This appendix specifies the planned trace of one pull request (PR) and the planned metrics, for a service that doesn't exist yet.

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
| `github.pull_request` | `root`, empty key | PR `created_at` | `merged_at` or `closed_at` | First merge or close |
| `github.pull_request.commit` | `commit`, commit hash | Author date | Committer date | When the commit appears on the PR |
| `github.pull_request.review_request` | `review_request`, with the timeline event's ID as key | The timeline event's `created_at` | The reviewer's next review, or the request's removal | At that end |
| `github.pull_request.review` | `review`, review node ID | Review `submitted_at` | Review `submitted_at` | When GitHub reports the review as submitted |
| `github.pull_request.merge` | `merge`, empty key | Last approving review, or PR creation without one | `merged_at` | At merge |
| `github.pull_request.close` | `close`, empty key | Last review, or PR creation without one | `closed_at` | At close without merge |

Each child's parent is the root span ID, which every run derives from `N` alone. A review span has zero duration on purpose: the wait before a review belongs to its review request span. A commit's span runs from authoring to committing, and for most commits that's zero or a few seconds.

## Root span events

Root events take GitHub's timestamps and don't need spans of their own.

- `ready_for_review` and `converted_to_draft`, from the PR timeline, the webhook action, or both.
- `review_thread.resolved` and `review_thread.unresolved`, from `pull_request_review_thread`.
- `head_ref_force_pushed`, from the PR timeline.
- `reopened`, from the timeline. After the first close, a reopened PR's later activity arrives as children and events, since the root already exists in the backend.

Review comments don't get a span or an attribute, because the service never reads comment text and a submitted review already marks the moment.

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

The `vcs.*` names follow the OTel semantic conventions for version control, which are still in development, so their exact names are unverified. Names under `github.*` are this service's own. Titles, bodies, and comment text never appear.

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
2. A milestone counts if GitHub's state shows it, the store has no record of it, and its end timestamp is less than 24 hours old.
3. The worker records the duration on the histogram and writes `(N, milestone)` to the store in the same step.
4. A milestone older than 24 hours gets a store record and no observation, so backfills and long outages never produce a spike of old values.

A crash after step 3 and before the next periodic export loses that observation, along with any other observation recorded since the last export. That window lasts 60 seconds at most. The design accepts it.

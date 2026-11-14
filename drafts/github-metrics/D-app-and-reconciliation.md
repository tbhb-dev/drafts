<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: GitHub App and reconciliation

Back to [the design](README.md). This appendix lists the planned App's events and permissions, then the reconciliation schedule and its rate-limit budget. Appendix A cites the GitHub documentation behind each number.

## Permissions

| Permission | Access | Why |
| --- | --- | --- |
| Metadata | Read | GitHub requires it of every App, and the reconciler needs repository metadata |
| Pull requests | Read | It covers the three subscribed events and every REST call the reconciler makes |

GitHub's permission tables put the list endpoints for pull requests (PRs), commits, reviews, and requested reviewers under Pull requests read. They list the issue timeline endpoint under either Issues read or Pull requests read, and that means the timeline works without the Issues permission. When the service requests an installation token, it asks for exactly these two permissions.

## Webhook events

| Event | Actions used | Needs | Why |
| --- | --- | --- | --- |
| `pull_request` | `opened`, `closed`, `reopened`, `synchronize`, `review_requested`, `review_request_removed`, `ready_for_review`, `converted_to_draft` | Pull requests read | Lifecycle changes and review requests, as a prompt to reconcile |
| `pull_request_review` | `submitted`, `dismissed` | Pull requests read | First review and approval milestones |
| `pull_request_review_thread` | `resolved`, `unresolved` | Pull requests read | Root span events for thread resolution |

The worker ignores every other action of these events, such as `edited` or `labeled`. Each App also receives `installation` and `installation_repositories` by default without subscribing. The service uses `installation_repositories` with action `added` to start a backfill of a newly added repository.

### Deliberately excluded

- **`pull_request_review_comment`:** each review comment belongs to a review, and the `pull_request_review` event already reports the submission.
- **`issue_comment`:** conversation comments don't affect the timing metrics, and the event would need the Issues permission.
- **`check_run` and `check_suite`:** continuous integration (CI) belongs to the organization collector. These events also need Checks read and arrive once per job.
- **`push`:** it needs Contents read, which grants access to source code. The PR commits endpoint gives the same commits under Pull requests read.
- **Issues, Actions, and workflow events:** later releases may add them.

### Manifest keys

The App manifest sets these two keys, with `public` set to `false` and the webhook URL `https://github-metrics.tbhb.dev/webhook` under `hook_attributes`.

```json
{
  "default_permissions": {
    "metadata": "read",
    "pull_requests": "read"
  },
  "default_events": [
    "pull_request",
    "pull_request_review",
    "pull_request_review_thread"
  ]
}
```

## Installation tokens

1. The service signs a JSON Web Token (JWT) with the App's private key, using RS256, with the App's client ID as `iss`, `iat` 60 seconds in the past, and `exp` 9 minutes ahead, inside GitHub's 10-minute limit.
2. It exchanges the JWT for an installation token that expires after one hour, and asks only for read access to metadata and pull requests.
3. It keeps the token in memory and renews it once less than 5 minutes remain. It never writes the token to disk or to a log.

## Reconciliation schedule

| Run | Interval | Scope |
| --- | --- | --- |
| Incremental | Every 15 minutes | Each repository's PRs, sorted by `updated` descending, until the repository's watermark |
| Open sweep | Nightly | Every open PR, to catch changes that don't move `updated_at` |
| Backfill | First start, and each repository added later | Every PR in the repository, oldest pages last, traces only |

A webhook schedules the same per-PR reconcile, delayed 30 seconds so a burst of events for one PR costs one reconcile. Each request sends the stored `ETag` in `If-None-Match`.

## Request budget

GitHub gives an App installation at least 5,000 requests an hour, and installations with more than 20 repositories get 50 more per repository, up to 12,500. Nobody has checked whether that counts every repository or only those past 20. The budget here uses the 5,000 floor.

| Work | Requests | Per hour, worst case with no `304` |
| --- | --- | --- |
| Incremental run: one repository list plus one PR list per repository | 41 | 164 |
| PRs changed, about 30 an hour: reviews, commits, and timeline | 3 each | 90 |
| Webhook reconciles, about 30 PRs an hour: the PR, then reviews, commits, and timeline | 4 each | 120 |
| Total steady state | | 374, about 8% of 5,000 |

The nightly sweep adds about 300 requests for 100 open PRs. Most return `304`, and GitHub doesn't count a conditional request that returns `304` against the primary limit, which puts the real cost well below the worst case. Whether `ETag` values change at each hourly token rotation is unverified, and the budget holds either way.

### Backfill

With 40 repositories of about 300 PRs each, a backfill reads 12,000 PRs. That's 120 list pages plus about 36,000 per-PR requests. The service throttles the backfill to 2,000 requests an hour, so it takes about 18 hours and never pushes total use past half the limit.

A backfill writes spans with their original timestamps and records metric milestones as seen, without histogram observations. A PR with more than 250 commits gets only its first 250 commit spans, because the PR commits endpoint returns at most 250.

### Rate-limit handling

- The service sends one request at a time, as GitHub advises, which keeps it far from the limit of 100 concurrent requests and 900 points a minute.
- It reads `x-ratelimit-remaining` and `x-ratelimit-reset` on every response. Backfill pauses while less than half the hourly limit remains, and all work waits for the reset when the remaining count reaches zero.
- On a secondary rate limit, the service waits for `retry-after` when present, and otherwise at least one minute, doubling up to 15 minutes.
- A `401` renews the installation token once, and a second `401` stops the run and logs the failure.

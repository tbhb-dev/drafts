<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: GitHub App and reconciliation

Back to [the design](README.md). This appendix lists the planned App's events and permissions, then the reconciliation schedule and its rate-limit budget. Appendix A cites the GitHub documentation behind each number.

## Permissions

| Permission | Access | Why |
| --- | --- | --- |
| Metadata | Read | GitHub requires it of every App, and the reconciler needs repository metadata |
| Pull requests | Read | It covers the two subscribed events and every REST call the reconciler makes |

GitHub's permission tables put the list endpoints for pull requests (PRs), commits, reviews, and requested reviewers under Pull requests read. They list the issue timeline endpoint under either Issues read or Pull requests read, and that means the timeline works without the Issues permission. When the service requests an installation token, it asks for exactly these two permissions.

Pull requests read reaches more than the service uses: PR titles, bodies, and review text, and PR diffs through the PR and PR files endpoints (existing, appendix A). The service doesn't call the diff or file endpoints, and README section "Data the service keeps and sends" states what it persists.

## Webhook events

| Event | Actions used | Needs | Why |
| --- | --- | --- | --- |
| `pull_request` | `opened`, `closed`, `reopened`, `synchronize`, `review_requested`, `review_request_removed`, `ready_for_review`, `converted_to_draft` | Pull requests read | Lifecycle changes and review requests, as a prompt to reconcile |
| `pull_request_review` | `submitted`, `dismissed` | Pull requests read | First review and approval milestones, as a prompt to reconcile |

The worker ignores every other action of these events, such as `edited` or `labeled`. Each App also receives `installation` and `installation_repositories` by default without subscribing. The service uses an `installation_repositories` body that lists `repositories_added` to start a backfill of each newly added repository.

### Deliberately excluded

- **`pull_request_review_thread`:** REST doesn't expose a thread's resolved state, so a missed delivery couldn't be healed, and the design takes no telemetry content from webhook bodies. Its payloads would also bring review comment text and diff hunks into the queue.
- **`pull_request_review_comment`:** each review comment belongs to a review, and the `pull_request_review` event already reports the submission.
- **`issue_comment`:** conversation comments don't affect the timing metrics, and the event would need the Issues permission.
- **`check_run` and `check_suite`:** continuous integration (CI) belongs to the organization collector. These events also need Checks read and arrive once per job.
- **`push`:** it needs Contents read, which grants access to a repository's full contents. The PR commits endpoint gives the same commits under Pull requests read.
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
    "pull_request_review"
  ]
}
```

## Installation tokens

1. The service signs a JSON Web Token (JWT) with the App's private key, using RS256, with the App's client ID as `iss`, `iat` 60 seconds in the past, and `exp` 9 minutes ahead, inside GitHub's 10-minute limit.
2. It exchanges the JWT for an installation token that expires after one hour, and asks only for read access to metadata and pull requests.
3. It keeps the token in memory and renews it once less than 5 minutes remain. It never writes the token to disk or to a log.

Retiring a previous webhook secret and rotating the App's private key are gates of the threat-model follow-up in README plan step 6.

## Reconciliation schedule

| Run | Interval | Scope |
| --- | --- | --- |
| Incremental | Every 15 minutes | Each repository's PRs, sorted by `updated` descending, until the repository's watermark |
| Open sweep | Nightly | Every open PR, to catch changes that don't move `updated_at` |
| Backfill | First start, and each repository added later | Every PR in the repository, oldest pages last, traces only |

A webhook schedules the same per-PR reconcile, delayed 30 seconds so a burst of events for one PR costs one reconcile. Each request sends the stored `ETag` in `If-None-Match`.

## Request budget

GitHub gives an App installation at least 5,000 requests an hour, and installations with more than 20 repositories get 50 more per repository, up to 12,500. Nobody has checked whether that counts every repository or only those past 20. The budget here uses the 5,000 floor.

### Pagination in the estimates

List endpoints return at most 100 items a page. The estimates assume each per-PR fetch of reviews, commits, and timeline averages 4 requests, not 3, because about a third of the lists need a second page, and a PR with more than 100 commits or timeline events needs more (assumption, unverified until the first run measures it).

An incremental run usually needs one PR list page per repository, because the walk stops at the watermark. The estimate adds 4 pages for busy repositories.

### Steady state

| Work | Requests | Per hour, worst case with no `304` |
| --- | --- | --- |
| Incremental run: one repository list, one PR list page per repository, and 4 extra pages | 45 | 180 |
| PRs changed, about 30 an hour: reviews, commits, and timeline with pagination | 4 each | 120 |
| Webhook reconciles, about 30 PRs an hour: the PR, then reviews, commits, and timeline | 5 each | 150 |
| Total steady state | | 450, about 9% of 5,000 |

The nightly sweep adds about 400 requests for 100 open PRs. Most return `304`, and GitHub doesn't count a conditional request that returns `304` against the primary limit, which puts the real cost well below the worst case. Whether `ETag` values change at each hourly token rotation is unverified, and the worst-case figures assume they do.

### One shared budget

Every request goes through one token bucket that the backfill, the nightly sweep, the incremental runs, and webhook-triggered reconciles all share. It refills continuously at one token every 1.44 seconds, which is 2,500 an hour, half of the 5,000 floor, and holds at most 60 tokens, so any rolling hour sees at most 2,560 requests. Every request takes one token, `304` answers included.

Each new token goes to the highest-priority waiter: webhook-triggered reconciles, then incremental runs, then the nightly sweep, then the backfill. The backfill always has work waiting, so it takes only the tokens nobody else wants, and webhook work never waits behind a spent hour.

That leaves the backfill about 2,500 − 450 = 2,050 requests in an ordinary hour, and about 2,500 − 450 − 400 = 1,650 in the nightly sweep's hour. The service also reads `x-ratelimit-remaining` on every response and stops all work until `x-ratelimit-reset` if fewer than 500 remain, which covers any other use of the installation's limit.

### Pacing for the secondary limit

The secondary limit is 900 points a minute for REST, and most `GET` requests cost 1 point. Sending one request at a time doesn't bound points per minute on its own, so the limiter also spaces requests at least 250 milliseconds apart, even while spending a full bucket. That allows at most 240 requests, and so at most 240 points, a minute.

Whether a `304` costs a secondary-limit point is unverified, so the pacer spaces it like any other request.

### Backfill

With 40 repositories of about 300 PRs each, a backfill reads 12,000 PRs. That's 120 list pages plus about 48,000 per-PR requests at 4 each. At about 2,050 requests an hour, it takes about 24 hours, and the token bucket keeps total use at or under about 2,560 in any rolling hour throughout.

A backfill writes spans with their original timestamps and records metric milestones as seen, without histogram observations. A PR with more than 250 commits gets only its first 250 commit spans, because the PR commits endpoint returns at most 250.

### Rate-limit handling

- The limiter sends one request at a time, at least 250 milliseconds apart, and only with a token from the shared bucket.
- It reads `x-ratelimit-remaining` and `x-ratelimit-reset` on every response, and stops all work until the reset when fewer than 500 remain.
- On a secondary rate limit, the service waits for `retry-after` when present, and otherwise at least one minute, doubling up to 15 minutes.
- A `401` renews the installation token once, and a second `401` stops the run and logs the failure.

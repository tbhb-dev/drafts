<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: evidence

Back to [the design](README.md).

Each claim has one label. Existing means a primary source states it, and the author read that source for this draft on 2026-11-13. New means this design proposes it, and unverified means nobody has checked it yet.

## GitHub documentation pages

Each page below resolved on 2026-11-13. A resolving page proves only that the page exists, and the claims table says which claims were read in it.

| Short name | URL |
| --- | --- |
| Webhook events and payloads | <https://docs.github.com/en/webhooks/webhook-events-and-payloads> |
| Validating webhook deliveries | <https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries> |
| Handling failed webhook deliveries | <https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries> |
| Permissions required for GitHub Apps | <https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps> |
| Rate limits for the REST API | <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api> |
| Best practices for using the REST API | <https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api> |
| Issue event types | <https://docs.github.com/en/rest/using-the-rest-api/issue-event-types> |
| Timeline endpoints | <https://docs.github.com/en/rest/issues/timeline> |
| Pull request endpoints | <https://docs.github.com/en/rest/pulls/pulls> |
| Review comment endpoints | <https://docs.github.com/en/rest/pulls/comments> |
| GitHub App webhook endpoints | <https://docs.github.com/en/rest/apps/webhooks> |
| GitHub App endpoints | <https://docs.github.com/en/rest/apps/apps> |
| Generating a JWT | <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-json-web-token-jwt-for-a-github-app> |
| Generating an installation token | <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app> |
| Registering an App from a manifest | <https://docs.github.com/en/apps/sharing-github-apps/registering-a-github-app-from-a-manifest> |

## GitHub

| Claim | Label | Source |
| --- | --- | --- |
| The `pull_request`, `pull_request_review`, `pull_request_review_comment`, and `pull_request_review_thread` events each need Pull requests read | Existing | Webhook events and payloads |
| `issue_comment` needs Issues read, `check_run` and `check_suite` need Checks read, and `push` needs Contents read | Existing | Webhook events and payloads |
| Every App receives `installation` and `installation_repositories` by default | Existing | Webhook events and payloads |
| GitHub caps webhook payloads at 25 MB | Existing, re-read 2026-11-13 | Webhook events and payloads |
| The webhook signature is an HMAC over the payload contents only, so headers such as `X-GitHub-Delivery` aren't covered | Existing, re-read 2026-11-13 | Validating webhook deliveries |
| A `pull_request_review_thread` payload carries no GitHub timestamp for the resolve or unresolve itself | Unverified | Webhook events and payloads, not re-read for this field |
| Listing pull requests (PRs), a PR's commits, its reviews, and its requested reviewers needs Pull requests read | Existing | Permissions required for GitHub Apps |
| The PR files endpoint is under Pull requests read | Existing, re-read 2026-11-13 | Permissions required for GitHub Apps |
| The PR endpoint can return a diff through the `application/vnd.github.diff` media type | Existing, re-read 2026-11-13 | Pull request endpoints |
| The issue timeline endpoint accepts either Issues read or Pull requests read | Existing | Permissions required for GitHub Apps |
| An installation gets at least 5,000 requests an hour, 50 more per repository once it has more than 20, and at most 12,500 | Existing | Rate limits for the REST API |
| Secondary limits include 100 concurrent requests and 900 points a minute for REST, and a client waits for `retry-after` or at least one minute | Existing, 900 points re-read 2026-11-13 | Rate limits for the REST API |
| Most REST `GET`, `HEAD`, and `OPTIONS` requests cost 1 secondary-limit point | Existing, re-read 2026-11-13 | Rate limits for the REST API |
| A `304` answer costs no secondary-limit point | Unverified | The pacer counts it anyway |
| A conditional request that returns `304` doesn't count against the primary limit when it includes an `Authorization` header | Existing | Best practices for using the REST API |
| GitHub advises serial requests to avoid secondary limits | Existing | Best practices for using the REST API |
| List endpoints return at most 100 items a page | Existing, re-read 2026-11-13 | Pull request endpoints, `per_page` |
| Timeline event types include `review_requested`, `ready_for_review`, `convert_to_draft`, `head_ref_force_pushed`, and `reopened` | Existing, re-read 2026-11-13 | Issue event types |
| A timeline `review_requested` event has `id`, `node_id`, and `created_at` | Existing | Issue event types |
| The REST review comment endpoints and timeline event types expose no thread resolution state, and review comments carry `diff_hunk` | Existing, re-read 2026-11-13: neither page lists a resolved field or event | Review comment endpoints, Issue event types |
| GitHub doesn't redeliver failed deliveries automatically | Existing | Handling failed webhook deliveries |
| Listing and redelivering App webhook deliveries needs a JSON Web Token (JWT) | Existing | GitHub App webhook endpoints |
| A JWT expires at most 10 minutes ahead, uses RS256, and should set `iat` 60 seconds in the past | Existing | Generating a JWT |
| An installation token lasts one hour, and its request can narrow its permissions | Existing | Generating an installation token |
| The PR commits endpoint returns at most 250 commits | Existing, re-read 2026-11-13 | Pull request endpoints |
| The manifest keys are `default_permissions` and `default_events` | Existing | Registering an App from a manifest |
| The permission keys are `pull_requests` and `metadata` | Existing | GitHub App endpoints, "Create an installation access token for an app." Its `permissions` body uses these names, and the manifest flow in plan step 7 confirms them |
| `ETag` values remain valid across the hourly token rotation | Unverified | The first live run after plan step 7 measures the `304` rate |
| A redelivery keeps its original `X-GitHub-Delivery` value | Unverified | The docs read for this draft don't say |
| `bbolt` doesn't zero freed pages, so logically deleted bodies can remain in the file | Unverified | Plan step 6 covers file modes and backups |
| About a third of per-PR lists need a second page | Unverified | An assumption for the budget, and the first run measures it |

## Telemetry libraries and backends

| Claim | Label | Source |
| --- | --- | --- |
| OpenTelemetry (OTel) treats an all-zero trace or span ID as invalid | Existing | W3C Trace Context and the OTel trace API specification |
| The Go SDK accepts explicit start and end timestamps on spans | Existing, not re-read for this draft | OTel Go `trace.WithTimestamp` option |
| A custom ID generator in the Go SDK can return the derived IDs | Unverified | Plan step 2 |
| The `vcs.*` attribute names match the current semantic conventions | Unverified | The conventions are in development, and step 3 pins a version |
| The backend shows and searches spans dated weeks in the past | Unverified | Plan step 2 |
| Backends keep or drop a second span with an already-exported ID | Unverified | Nobody has tested it, and the design avoids re-exporting the root |
| The host collector accepts OTel Protocol (OTLP) over HTTP on port 4318 | Unverified | Port 4318 is the OTLP default, and step 2 confirms it on the host |
| Who can read the backends, and how long they keep this telemetry | Unverified | The operator confirms it in the threat-model follow-up, plan step 6 |

## This design

| Claim | Label | Source |
| --- | --- | --- |
| tbhb-dev has tens of repositories, about 40 | Unverified | An assumption for the budget, and the first run counts them |
| About 30 PRs change an hour at peak | Unverified | An assumption for the budget |
| An organization collector design covers continuous integration (CI), Actions, and organization-level counters | Unverified | This draft didn't read it, and its owner confirms the boundary in plan step 4 |
| This repository can't render D2 diagrams | Existing | `MODULE.bazel` on `main` has no D2 module |

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: evidence

Back to [the design](README.md).

Each claim has one label. Existing means a primary source states it, and the author read that source for this draft on 2026-11-13. New means this design proposes it, and unverified means nobody has checked it yet.

## GitHub

| Claim | Label | Source |
| --- | --- | --- |
| The `pull_request`, `pull_request_review`, `pull_request_review_comment`, and `pull_request_review_thread` events each need Pull requests read | Existing | GitHub Docs, "Webhook events and payloads" |
| `issue_comment` needs Issues read, `check_run` and `check_suite` need Checks read, and `push` needs Contents read | Existing | GitHub Docs, "Webhook events and payloads" |
| Every App receives `installation` and `installation_repositories` by default | Existing | GitHub Docs, "Webhook events and payloads" |
| GitHub caps webhook payloads at 25 MB | Existing | GitHub Docs, "Webhook events and payloads" |
| Listing pull requests (PRs), a PR's commits, its reviews, and its requested reviewers needs Pull requests read | Existing | GitHub Docs, "Permissions required for GitHub Apps" |
| The issue timeline endpoint accepts either Issues read or Pull requests read | Existing | GitHub Docs, "Permissions required for GitHub Apps" |
| An installation gets at least 5,000 requests an hour, 50 more per repository once it has more than 20, and at most 12,500 | Existing | GitHub Docs, "Rate limits for the REST API" |
| Secondary limits include 100 concurrent requests and 900 points a minute for REST, and a client waits for `retry-after` or at least one minute | Existing | GitHub Docs, "Rate limits for the REST API" |
| A conditional request that returns `304` doesn't count against the primary limit when it includes an `Authorization` header | Existing | GitHub Docs, "Best practices for using the REST API" |
| GitHub advises serial requests to avoid secondary limits | Existing | GitHub Docs, "Best practices for using the REST API" |
| A timeline `review_requested` event has `id`, `node_id`, and `created_at` | Existing | GitHub Docs, "Timeline events" |
| GitHub doesn't redeliver failed deliveries automatically | Existing | GitHub Docs, "Handling failed webhook deliveries" |
| Listing and redelivering App webhook deliveries needs a JSON Web Token (JWT) | Existing | GitHub Docs, "REST API endpoints for GitHub App webhooks" |
| A JWT expires at most 10 minutes ahead, uses RS256, and should set `iat` 60 seconds in the past | Existing | GitHub Docs, "Generating a JSON Web Token (JWT) for a GitHub App" |
| An installation token lasts one hour, and its request can narrow its permissions | Existing | GitHub Docs, "Generating an installation access token for a GitHub App" |
| The PR commits endpoint returns at most 250 commits | Existing | GitHub Docs, "REST API endpoints for pull requests" |
| The manifest keys are `default_permissions` and `default_events` | Existing | GitHub Docs, "Registering a GitHub App from a manifest" |
| The permission keys are `pull_requests` and `metadata` | Existing | The GitHub Docs section "Create an installation access token for an app." Its `permissions` body uses these names, and the manifest flow in plan step 6 confirms them |
| `ETag` values remain valid across the hourly token rotation | Unverified | The first live run after plan step 6 measures the `304` rate |
| A redelivery keeps its original `X-GitHub-Delivery` value | Unverified | The docs read for this draft don't say |

## Telemetry libraries and backends

| Claim | Label | Source |
| --- | --- | --- |
| OpenTelemetry (OTel) treats an all-zero trace or span ID as invalid | Existing | W3C Trace Context and the OTel trace API specification |
| The Go SDK accepts explicit start and end timestamps on spans | Existing, not re-read for this draft | OTel Go `trace.WithTimestamp` option |
| A custom ID generator in the Go SDK can return the derived IDs | Unverified | Plan step 2 |
| The `vcs.*` attribute names match the current semantic conventions | Unverified | The conventions are in development, and step 3 pins a version |
| The backend shows and searches spans dated weeks in the past | Unverified | Plan step 2 |
| The host collector accepts OTel Protocol (OTLP) over HTTP on port 4318 | Unverified | Port 4318 is the OTLP default, and step 2 confirms it on the host |

## This design

| Claim | Label | Source |
| --- | --- | --- |
| tbhb-dev has tens of repositories, about 40 | Unverified | An assumption for the budget, and the first run counts them |
| About 30 PRs change an hour at peak | Unverified | An assumption for the budget |
| An organization collector design covers continuous integration (CI), Actions, and organization-level counters | Unverified | This draft didn't read it, and its owner confirms the boundary in plan step 4 |
| This repository can't render D2 diagrams | Existing | `MODULE.bazel` on `main` has no D2 module |

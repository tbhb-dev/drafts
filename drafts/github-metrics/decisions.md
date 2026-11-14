<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: decisions

Back to [the design](README.md). Each item keeps its number for life. An answered item moves from Open to Decided as it stands.

## Open

None.

## Decided

The operator decided D1 and D2 on 2026-11-14, and each item states its record. Platform engineering decided D3 through D13 on 2026-11-14, each one is a two-way door, and this design's pull request in tbhb-dev/wiki is the record for each.

### D1. May tbhb-dev create the read-only GitHub App and install it on all repositories?

- **Decided:** 2026-11-14, by the operator: yes. A later answer the same day widened the permissions to every read-only permission GitHub offers, for later telemetry. It never includes write.
- **Rule:** the App has every read-only permission for repositories and the organization, and no write permission. It can read source code, issues, checks, and PR data from every repository, private ones included, and its private key is a new credential. The service requests installation tokens narrowed to metadata and pull requests.
- **Changed afterwards by platform engineering:** the App subscribes to two PR events instead of three, because D11 drops `pull_request_review_thread`.
- **Undo:** uninstall or delete the App, which revokes every token at once. To narrow it, edit the App's permissions, and every installation must accept the change.
- **Record:** the operator's answer in chat, 2026-11-14.

### D2. May the webhook path face the public internet through a Cloudflare Tunnel at `github-metrics.tbhb.dev`, and may the image be public?

- **Decided:** 2026-11-14, by the operator: yes. A later answer the same day also allows publishing the container image to the public GitHub Container Registry.
- **Rule:** the tunnel routes only `POST /webhook`, and the edge answers `404` to every other path. The tunnel makes outbound connections only, and the service checks each request's signature before it parses the body.
- **Undo:** remove the tunnel route and the DNS record, and unpublish or delete the image package.
- **Record:** the operator's answer in chat, 2026-11-14.

### D3. Trace and span IDs derive from the PR's GraphQL node ID

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the trace ID is the first 16 bytes of a Secure Hash Algorithm (SHA) SHA-256 digest over a versioned prefix and the node ID. Span IDs come from hashing the node ID with the span kind and a key unique within the PR. The service accepts the collision odds in appendix C and doesn't check for collisions.
- **Undo:** change the version prefix, which moves every later span to new IDs.
- **Record:** this design's pull request.

### D4. The service stores its state in a small `bbolt` file on a mounted volume

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the service writes its whole state to one `bbolt` file on a writable volume. That covers the queue, the dedupe table, the exported span IDs, the metric milestones, the watermarks, the entity tags, and the store's creation time. The rest of the container is read-only. Restoring an older copy of the file is forbidden, and the operator deletes a damaged file so the service starts empty.
- **Undo:** swap the store behind its interface, such as for SQLite. Losing the file re-exports every span of every PR in scope once, through a fresh backfill, and the creation time keeps metrics from doubling.
- **Record:** this design's pull request.

### D5. Child spans export when complete, and the root exports once at the first merge or close

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** each child span goes out once its end time exists. The root goes out exactly once, at the PR's first terminal state, even after a reopen. After a reopen, the reopen, later closes and merges, and later root events become child spans, and the collector pipeline doesn't tail-sample.
- **Undo:** delay every span until the terminal state, which hides open PRs from the backend.
- **Record:** this design's pull request.

### D6. Webhooks answer 2xx after verification, dedupe, and a synced enqueue

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the receiver touches the dedupe table only after it verifies the signature. One transaction drops a known body hash, rejects a known delivery ID with a different body hash, and otherwise records both and enqueues the body, and the receiver answers `202` after the sync. A delivery only triggers a reconcile, and a delivery the worker can't parse moves to a dead-letter bucket after five tries.
- **Undo:** answer after an in-memory enqueue, which trades durability for latency.
- **Record:** this design's pull request.

### D7. Reconciliation is the only healer

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the first release doesn't call GitHub's redelivery API. A 15-minute incremental run and a nightly pass over open PRs heal missed deliveries, which works because no telemetry depends on a webhook-only fact.
- **Undo:** add a redelivery client that signs a JSON Web Token (JWT) with the App key.
- **Record:** this design's pull request.

### D8. The service reads GitHub through REST with conditional requests and one shared budget

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the service calls the REST API with `If-None-Match` and sends one request at a time, at least 250 milliseconds after the previous one. One shared token bucket, refilled at 2,500 requests an hour with at most 60 tokens held, serves webhook-triggered reconciles first, then incremental runs, the nightly sweep, and the backfill.
- **Undo:** move the per-PR fetch to one GraphQL query, as appendix B describes.
- **Record:** this design's pull request.

### D9. Export over HTTP with protobuf to the host's collector

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the service exports OpenTelemetry Protocol (OTLP) data over HTTP to port 4318 and exports trace spans synchronously per reconcile, without the batch processor. Metrics leave through a periodic reader every 60 seconds.
- **Undo:** switch the exporter to gRPC on port 4317, which changes the configuration and the dependencies.
- **Record:** this design's pull request.

### D10. Repository and author type are the only metric attributes

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the metric names, units, and bucket boundaries match the README. The PR number and the author's login never become metric attributes, and only milestones less than 24 hours old and later than the store's creation time count. The worker commits a milestone before it records the observation.
- **Undo:** rename or add instruments in a later release. Old series stop, and new ones start empty.
- **Record:** this design's pull request.

### D11. Checks, issue comments, pushes, and review threads stay out of the first release

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the App has read access to Checks, Issues, and Contents but the service doesn't call them. It doesn't subscribe to `check_run`, `check_suite`, `issue_comment`, or `push`, nor to `pull_request_review_comment`, which a submitted review already signals, nor to `pull_request_review_thread`, whose resolution REST can't confirm.
- **Undo:** subscribe to the event, and the permission is already in place.
- **Record:** this design's pull request.

### D12. Telemetry contains no PR titles, bodies, comment text, or diffs

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** spans record repository and base branch names, PR numbers, logins, team slugs, states, and times, for private repositories too. Raw webhook bodies, which contain titles and review text, stay in the store only until the worker extracts the repository and PR, or 7 days in the dead-letter bucket. The operator sets backend access and retention for this telemetry, and the threat-model follow-up must confirm them.
- **Undo:** add a title attribute in a later release.
- **Record:** this design's pull request.

### D13. The boundary with the organization collector, pending its owner

- **Decided:** 2026-11-14, by platform engineering, as an unverified assumption about the other design.
- **Rule:** `github-metrics` covers PRs, reviews, review requests, and commits on PRs. The organization collector covers CI, Actions, and organization-level counters, and each uses its own App or token.
- **Undo:** the other design's owner moves the line, and this design follows.
- **Record:** this design's pull request.

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: decisions

Back to [the design](README.md). Each item keeps its number for life. An answered item moves from Open to Decided as it stands.

## Open

### D1. May tbhb-dev create the read-only GitHub App and install it on all repositories?

- **Recommendation:** yes, with exactly two permissions, read access to pull requests and to metadata, and the three pull request (PR) events that appendix D lists.
- **Why:** the App reads PR, review, and commit data from every repository, private ones included, and its private key is a new credential. The App can't read source code, issues, or continuous integration (CI) results with these permissions.
- **If yes:** platform engineering generates the manifest, and an organization owner completes GitHub's manifest flow. The private key and webhook secret then go into 1Password for the deploy host's secret manager.
- **If no:** the service can't run as designed. A narrower install on selected repositories works the same way and covers less.
- **Reversible:** yes. Uninstalling the App or deleting it revokes every token at once, and deleting a private key revokes that key.

### D2. May the webhook path face the public internet through a Cloudflare Tunnel at github-metrics.tbhb.dev?

- **Recommendation:** yes, with the tunnel route limited to `POST /webhook` and every other path answered `404` at the edge.
- **Why:** GitHub delivers webhooks from the internet, and a public endpoint on this host loosens its trust boundary. The tunnel makes outbound connections only, and the service checks each request's signature before it parses the body.
- **If yes:** platform engineering adds the DNS record and the tunnel route with the deployment in plan step 6.
- **If no:** the service runs with reconciliation only, polling every 15 minutes. Traces and metrics then lag GitHub by up to 15 minutes, with the same content.
- **Reversible:** yes. Removing the tunnel route and the DNS record closes the path, and the App's webhook can switch off in its settings.

## Decided

Platform engineering made each of these decisions on 2026-11-14. Each one is a two-way door, and this design's pull request in tbhb-dev/wiki is the record for each.

### D3. Trace and span IDs derive from the PR's GraphQL node ID

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the trace ID is the first 16 bytes of a Secure Hash Algorithm (SHA) SHA-256 digest over a versioned prefix and the node ID. Span IDs come from hashing the node ID with the span kind and a key unique within the PR. The service accepts the collision odds in appendix C and doesn't check for collisions.
- **Undo:** change the version prefix, which moves every later span to new IDs.
- **Record:** this design's pull request.

### D4. The service stores its state in a small `bbolt` file on a mounted volume

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** one `bbolt` file on a writable volume stores the queue and the exported span IDs. It also stores metric milestones, watermarks, delivery IDs, and entity tags. The rest of the container is read-only.
- **Undo:** swap the store behind its interface, such as for SQLite. Losing the file costs at most one duplicate span per PR and a fresh backfill without metrics.
- **Record:** this design's pull request.

### D5. Child spans export when complete, and the root exports at merge or close

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** each child span goes out once its end time exists. The root goes out once, at the PR's first terminal state, and the collector pipeline doesn't tail-sample.
- **Undo:** delay every span until the terminal state, which hides open PRs from the backend.
- **Record:** this design's pull request.

### D6. Webhooks answer 2xx after verification and a synced enqueue

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the receiver answers `202` only after it verifies the signature and syncs the delivery to the store. A delivery the worker can't parse moves to a dead-letter bucket after five tries, and a PR whose reconcile fails retries until it succeeds.
- **Undo:** answer after an in-memory enqueue, which trades durability for latency.
- **Record:** this design's pull request.

### D7. Reconciliation is the only healer

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the first release doesn't call GitHub's redelivery API. A 15-minute incremental run and a nightly pass over open PRs heal missed deliveries.
- **Undo:** add a redelivery client that signs a JSON Web Token (JWT) with the App key.
- **Record:** this design's pull request.

### D8. The service reads GitHub through REST with conditional requests

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the service calls the REST API with `If-None-Match`, one request at a time. Its use stays under half of the installation's hourly limit.
- **Undo:** move the per-PR fetch to one GraphQL query, as appendix B describes.
- **Record:** this design's pull request.

### D9. Export over HTTP with protobuf to the host's collector

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the service exports OpenTelemetry Protocol (OTLP) data over HTTP to port 4318 and exports trace spans synchronously per reconcile, without the batch processor.
- **Undo:** switch the exporter to gRPC on port 4317, which changes the configuration and the dependencies.
- **Record:** this design's pull request.

### D10. Repository and author type are the only metric attributes

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the metric names, units, and bucket boundaries match the README. The PR number and the author's login never become metric attributes, and only milestones less than 24 hours old count.
- **Undo:** rename or add instruments in a later release. Old series stop, and new ones start empty.
- **Record:** this design's pull request.

### D11. Checks, issue comments, and pushes stay out of the first release

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** the App doesn't request the Checks, Issues, or Contents permissions. It doesn't subscribe to `check_run`, `check_suite`, `issue_comment`, or `push`, nor to `pull_request_review_comment`, which a submitted review already signals.
- **Undo:** add the permission and the event, which every installation must accept again.
- **Record:** this design's pull request.

### D12. Telemetry contains no PR titles, bodies, or comment text

- **Decided:** 2026-11-14, by platform engineering.
- **Rule:** spans record the repository, the PR number, logins, states, and times. Titles, bodies, and comment text from private repositories never leave GitHub through this service.
- **Undo:** add a title attribute in a later release.
- **Record:** this design's pull request.

### D13. The boundary with the organization collector, pending its owner

- **Decided:** 2026-11-14, by platform engineering, as an unverified assumption about the other design.
- **Rule:** `github-metrics` covers PRs, reviews, review requests, and commits on PRs. The organization collector covers CI, Actions, and organization-level counters, and each uses its own App or token.
- **Undo:** the other design's owner moves the line, and this design follows.
- **Record:** this design's pull request.

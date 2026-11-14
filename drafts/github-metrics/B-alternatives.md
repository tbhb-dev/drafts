<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# GitHub metrics for pull requests: alternatives considered

Back to [the design](README.md). Each section describes one option not taken and the reason.

## A stateless service

The service could re-derive everything from GitHub on each pass and not keep any state. OpenTelemetry (OTel) backends don't reliably drop a span exported twice, and a histogram counts every observation, so each pass would double the metrics. A mounted volume with one `bbolt` file needs a few megabytes and keeps the root filesystem read-only.

## SQLite instead of `bbolt`

SQLite gives queries and a familiar format. A static binary with cgo off needs a pure Go SQLite port, which adds a large dependency for what amounts to a few key-value buckets. `bbolt` is pure Go, small, and transactional, and an interface in front of the store allows a swap later.

## One complete trace at merge or close

Waiting until the pull request (PR) ends to export any span gives backends whole traces and suits tail sampling. It hides open PRs for weeks, though those are the ones a reader most wants to inspect, and it needs the store to keep each pending span. Incremental children with a late root show progress at the cost of a missing root while the PR is open.

## Random IDs with a lookup table

Random trace and span IDs stored per PR would also give one trace per PR. They make the store the only source of the IDs, so losing the file splits every open PR's trace in two. Derived IDs come out the same after a lost store, and webhook and reconciliation runs agree without coordination.

## One query per pull request

One GraphQL query can fetch a PR with its reviews, commits, and timeline, which saves requests. GraphQL has its own point-based limit and no conditional requests, and every poll costs points even when nothing changed. REST with `If-None-Match` makes the common idle poll free, and the request asked for REST.

## A streaming export protocol

OTel Protocol (OTLP) over gRPC streams well and is the collector's other default. At a few hundred spans an hour, streaming buys nothing, and gRPC adds a large dependency tree to the binary. Debugging HTTP with protobuf takes less tooling, and switching later is a configuration change.

## The collector's own GitHub receiver

The OTel Collector has a contributed GitHub receiver that scrapes repository metrics and can take webhooks. Its metrics are per-change gauges, not histograms per repository, and it builds no deterministic PR lifecycle trace, as far as this draft knows, which is unverified. A small dedicated service fits the trace model and the idempotency rules better.

## The redelivery API as a healer

The service could list failed deliveries and redeliver them with an App-level JSON Web Token (JWT). Reconciliation already reads the PR's current state and heals the same gaps. A second healing path adds code and a second credential use for little gain.

## Author login as a metric attribute

Per-author histograms would show who waits longest. Public repositories accept PRs from anyone, so the login set has no bound, and per-person timing invites a use nobody asked for. Traces still record the author of each PR.

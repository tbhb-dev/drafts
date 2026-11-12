# Agent capacity container image

Draft of 2026-11-12. It proposes a container image for `agent-capacity`. The proposal describes future work, and the project hasn't published an image. The operator has one open question, in the last section.

## Scope

The `serve` command is the collector. It reads files and exports OpenTelemetry metrics, so it fits a container. Claude Code starts `statusline` natively on its own machine. For the keeper, native execution provides access to the sign-in directories it needs to drive interactive panes. Because each machine exports its own series, the image doesn't need a sync path.

## Image

The image contains one static Go binary with cgo off and runs it as a non-root user. It targets `linux/amd64` and `linux/arm64`. A distroless static base pinned by digest brings the CA certificates the OpenTelemetry Protocol (OTLP) exporter needs for Transport Layer Security (TLS). A `scratch` base would work only if the binary embedded its own root certificates, which adds a dependency.

Go rules in this organization (`tbhb_rules_go`) export lint, coverage, mutation, vulnerability, and module checks. They don't export image rules. `go/defs.bzl` in the rules repository confirms this. The Bazel Central Registry lists `rules_oci`, which builds images without a Docker daemon. Whether it fits the lockfile and hermetic settings here is unverified. A spike decides it before the image PR, and the fallback is an image assembled in CI from the Bazel-built binaries.

## Mounts and credentials

`serve` reads two places. A spool directory stores the snapshots the `statusline` command writes, and the Codex sessions directory holds rollout files. Both mount read-only. By default, the image sets `AGENT_CAPACITY_SPOOL_DIR` to `/spool` and `CODEX_HOME` to `/codex`. The container runs with a user ID that can read the spool, which is mode 0700.

The image and mounts exclude sign-in and token directories. They also exclude configuration directories. Standard `OTEL_EXPORTER_OTLP_*` variables supply the OTLP destination at run time.

## CI

The Bazel job runs on `ubuntu-26.04` and `ubuntu-26.04-arm`. It also runs on `macos-26`. The macOS leg stays because `statusline` runs natively on Macs. An image job on Linux builds both architectures on every pull request and pushes nothing. Pushing waits for the operator's answer below.

## Open question for the operator

Where may `agent-capacity` publish container images? This would be the first image tbhb-dev publishes.

Recommendation: yes, to `ghcr.io`, as a public package of the public repository.

- **Registry:** GitHub's container registry. It's free for public packages. This is unverified against the current terms.
- **Visibility:** public, to match the repository.
- **CI token:** the workflow's own `GITHUB_TOKEN` with `packages: write`, granted only to the push job that runs on `main`. No stored credential.
- **Pinning:** deploys reference the image by digest. Tags are labels and never what a host runs.
- **Withdrawal:** delete the package version through the registry API, then redeploy the previous digest. A host pinned to a deleted digest fails to pull, which is the intended signal.

Until the operator answers, CI builds the image and doesn't push it.

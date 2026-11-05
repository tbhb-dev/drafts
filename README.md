# Drafts

This public repository holds working design drafts for tbhb-dev work. A draft is an input to a change in another repository, not published documentation. It contains only new, clean work. A private counterpart exists for drafts that can't be public.

## Layout

Each draft is a directory under `drafts/`, named for its subject. Its main document is `README.md`, beside any supporting files.

## Workflow

Every draft lands through a pull request. See [AGENTS.md](AGENTS.md) for the setup and the rules.

## Checks

Bazel runs every gate from tbhb-dev/rules: `bazelisk test //...`. The spelling, style, and Markdown checks read each draft.

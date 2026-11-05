# Agent instructions

This is tbhb-dev/drafts, the public repository of working design drafts for tbhb-dev work. It's public and contains only new, clean work. A private counterpart exists for drafts that can't be public. Never copy content from it into this repository.

## Standing rules

- Everything here is public. Never write a private repository's name, a private host name, tailnet details, an internal URL, or anything from private work.
- Never write a public IP address in a commit, a pull request, or a draft. The `public-ipv4` gitleaks rule enforces this.
- Secrets live in 1Password only. Never paste one into chat or commit one to git.
- A draft informs a change elsewhere. The final docs go in the repository that makes the change.

## Writing a draft

Put each draft in its own directory under `drafts/`, with the main document as `README.md`. Keep only configuration at the repository root. Title its directory after the topic.

- A draft isn't an adopted decision. Adoption belongs to the session of the repository it concerns.
- Keep each qualification beside the claim it qualifies: draft, open question, undecided, decided by the operator, unverified.
- Describe the intended system directly when drafting architecture, keep unresolved choices explicit, and never claim a proposed feature exists today.
- Leave an earlier draft as a record of its date. Write a new one rather than rewriting its history.
- Write each Markdown paragraph on one line.

## Commits and pull requests

- Branch first. Every change lands through a pull request.
- Use Conventional Commits and a plain `git commit`, without `Signed-off-by` or `Co-authored-by`, and fill every section of `.github/PULL_REQUEST_TEMPLATE.md`.

## Checks

Run `mise install` and `bazelisk run //:bazel_env`, which builds the `.bazel_env/bin` shims the git hooks call, then `prek install`. Every gate is a Bazel test from tbhb-dev/rules, which `bazelisk test //...` runs, and `bazelisk run //:format` rewrites files into their canonical format. The pre-commit hooks run the same gates. To build against a local rules clone, put `common --override_module=tbhb_rules=<path to a rules clone>` in `.bazelrc.user`.

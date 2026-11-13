<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster writer checklist

Follow this list before opening a PR against [README.md](README.md). Use the row formats in [TEMPLATE.md](TEMPLATE.md).

## Before writing

- [ ] The evidence shows the model succeeded or failed at the task. A mixed or partial result isn't an observation yet.
- [ ] The task matches a task type in the task-fit table. A new task type needs a new task-fit row in the same PR.
- [ ] If the model has no roster row yet, add one first.

## Each observation

- [ ] The date is the day the work ran, in `yyyy-mm-dd` form.
- [ ] The session column holds a generic role, such as Coordinator or Reviewer, and never a session name.
- [ ] The outcome starts with Succeeded or Failed and says what the model did in one or two sentences.
- [ ] Cite a public commit or PR as owner/repo#N. Otherwise write `Internal record, yyyy-mm-dd, task type, outcome`, and never a private path, repository, or PR number.
- [ ] The outcome says who spot-checked the work and how, such as three commits checked.
- [ ] The row goes at the top of the table.

## Task-fit and roster changes

- [ ] A task-fit change rests on a ruling or on repeated observations, and its evidence column cites them.
- [ ] A claim without evidence says `unverified`.
- [ ] A row never contradicts a standing ruling by @tbhb.
- [ ] A cost cell says subscription quota or API spend. API spend cites @tbhb's approval and its budget.

## Privacy and style

- [ ] The change has no secrets, public IP addresses, host names, tailnet details, internal paths, private repository names, account addresses, or session names.
- [ ] Each Markdown paragraph is on one line.
- [ ] `vale`, `rumdl check`, and `gitleaks dir . --no-banner` pass on the changed files, and the outgoing private-name scan passes before the push.

## The PR

- [ ] One PR per session per day carries every observation from that day.
- [ ] The PR body fills every section of the PR template.

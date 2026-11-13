<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster

This page says which models tbhb-dev runs and what each suits. Read it before picking a model for a session, an agent, or a peer run. State the model by name on every agent call.

## Standing rulings

- Planning and design run on Opus 5.5 or Fable 5.2 at medium effort, never on Sonnet.
- Sonnet is the default model for other sessions, and it stays the fallback.
- Haiku is never the reviewer of record.
- Codex is the default harness for peer agents. Claude runs a peer agent only for a listed gap, such as an OS-sandbox spike.
- Codex `gpt-6.1-sol` is the second model family in class D, K, and guard reviews.
- Any API spend needs @tbhb's approval first.

## Models

| Model | Use for | Don't use for |
| --- | --- | --- |
| Opus 5.5 | Planning, design, and proposals | Routine edits that a cheaper model handles |
| Fable 5.2 | The same work as Opus 5.5 | Routine edits that a cheaper model handles |
| Sonnet | Script fixes, read-only tracker checks, search with citations, and mechanical edits | Planning and design |
| Haiku | A first pass over logs, classification, and mechanical checks that a person or Sonnet confirms | Review, checks where a miss matters, and deciding whether a change is on main from log subjects alone |
| Codex `gpt-6.1-sol` | Peer agents, builds, and second-family code review | Unverified |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token. Each has limits, and the shared Claude sign-in has hit its 5-hour limit. No one has verified the exact quota for each model.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet. The approver model spike decides the approver verdicts row.

## Updating this page

Change a recommendation only when [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling supports it, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).

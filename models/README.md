<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster

This page says which models tbhb-dev runs and what each suits. Read it before picking a model for a session, an agent, or a peer run. State the model by name on every agent call.

## Standing rulings

- Planning and design run on Opus 5.5 or Fable 5.2 at medium effort, never on Sonnet.
- Sonnet is the default model for Claude Code sessions and Claude subagents, and it stays their fallback.
- Haiku is never the reviewer of record.
- Codex is the default harness for peer agents. Claude runs a peer agent only where Codex can't, such as work that needs an operating-system sandbox.
- Codex is the default for building. Claude builds only for a gap that the Models table lists.
- Codex `gpt-6.1-sol` is the second model family for code review. Codex and Antigravity are full peers for the review read, and the reviewer of record checks every finding. Claude isn't the default review model, only whoever signs.
- Any API spend needs @tbhb's approval first.

## Models

| Job | Models and harnesses | Don't use |
| --- | --- | --- |
| Planning and design | Opus 5.5 or Fable 5.2 at medium effort | Sonnet or Haiku |
| Building | Codex `gpt-6.1-sol` by default. Claude builds only where Codex can't, such as work that needs an operating-system sandbox. When Claude builds, it runs on Sonnet | Opus 5.5 or Fable 5.2 for routine edits that a cheaper model handles |
| Code review | Codex `gpt-6.1-sol` and Antigravity `gemini-4-argon-high` as full peers for the review read. How well Antigravity reviews is unverified. The reviewer of record checks every finding and signs | Haiku as the reviewer of record |
| Read-only checks and scans | Antigravity `gemini-4-argon-high` first while it has quota, with its results unverified. Sonnet for tracker checks against main. Haiku for log scans that a person or Sonnet confirms | Haiku alone on checks where a miss matters |
| Coordination sessions | Claude Code on Sonnet, its default. Codex and Antigravity are unverified for this job | Sonnet for planning or design inside the session |
| First-pass triage | Haiku, and a person or Sonnet confirms its result. Antigravity `gemini-4-argon-high` first while it has quota, unverified | Haiku deciding whether a change is on main from commit subjects alone |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token, and how Antigravity bills is unverified. Each has limits, and the shared Claude sign-in has hit its 5-hour limit. No one has verified the exact quota for each model.

Antigravity also offers Gemini 4 Carbon (low, medium, and high), Gemini 4 Hydrogen (low through extra high), and two Orchestrator keys. They're available but untested, so their fit is unverified. `gemini-4-argon-high` stays the recommended Antigravity model until observations say otherwise.

## Routing by quota

Pick the harness by its remaining weekly headroom. Send read-only work, such as reviews, log scans, state checks, and triage, to the harness with the most headroom. Antigravity takes read-only peer work first while it has quota. Keep builds on Codex while it has headroom. Every Claude subagent call states its model, and that model is Sonnet or Haiku.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet.

## Updating this page

Change a recommendation only when [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling supports it, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).

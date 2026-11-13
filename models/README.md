<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster

This page is current guidance for choosing a model. Read it before picking a model for a session, an agent, or a peer run. Add an observation when a model plainly succeeds or fails at a task type. Every agent call states its model by name.

Each claim cites its source. That's a ruling by @tbhb, a public commit or PR, or an internal record with its date, task type, and outcome. Each fit claim without evidence says `unverified`.

## Standing rulings

These rulings from @tbhb override any observation below.

- Planning runs on Opus 5.5 or Fable 5.2 at medium effort, never on Sonnet. (Internal record, 2026-11-12, ruling by @tbhb)
- Other sessions run on Sonnet.
- Codex is the default harness for peer agents. Claude runs a peer agent only for a listed gap, such as an OS-sandbox spike.
- Codex `gpt-6.1-sol` is the second model family in class D, K, and guard reviews.
- Every agent call states its model.

## Roster

| Model | Vendor and harness | Account or billing | Cost to tbhb-dev | Limits |
| --- | --- | --- | --- | --- |
| Opus 5.5 | Anthropic, Claude Code | The shared Claude subscription sign-in | Subscription quota | A 5-hour limit, see the note |
| Fable 5.2 | Anthropic, Claude Code | Same sign-in as Opus 5.5 | Subscription quota, shared with Opus 5.5 | Per-model limits unverified |
| Sonnet | Anthropic, Claude Code | Same sign-in as Opus 5.5 | Subscription quota, shared with Opus 5.5 | Per-model limits unverified |
| Haiku | Anthropic, Claude Code | Same sign-in as Opus 5.5 | Subscription quota, shared with Opus 5.5 | Per-model limits unverified |
| Codex `gpt-6.1-sol` | OpenAI, Codex | A Codex subscription sign-in | Subscription quota | Unverified |

Claude and Codex both run on subscription quota, not per-token billing. The shared sign-in hit its 5-hour limit at 2026-11-12 03:15 Coordinated Universal Time (internal record, 2026-11-12), which shows it has limits. Exact quotas for each model aren't known.

The approver model spike holds the only approved API spend, a budget of 25 United States dollars. Any other API key or per-token billing needs @tbhb's approval first.

## Task fit

| Task type | Recommended | Acceptable fallback | Don't use | Evidence |
| --- | --- | --- | --- | --- |
| Search-and-cite | Haiku | Sonnet (unverified) | Unverified | Observation 2026-11-13, Haiku, search-and-cite |
| Script fix | Sonnet | Unverified | Unverified | Observation 2026-11-13, Sonnet, script fix |
| Code review, first family | Unverified | Unverified | Unverified | Unverified |
| Code review, second family | Codex `gpt-6.1-sol` | Unverified | Unverified | Standing ruling, and observation 2026-11-13, Codex, second-family review |
| Design or planning | Opus 5.5 or Fable 5.2, medium effort | The other of the two | Sonnet | Standing ruling (internal record, 2026-11-12, ruling by @tbhb) |
| Host ops | Unverified | Unverified | Unverified | Unverified |
| Console work | Unverified | Unverified | Unverified | Unverified |
| Grading evaluations | Unverified | Unverified | Unverified | Unverified |
| Approver verdicts | Pending | Pending | Pending | Pending the approver model spike, which compares `Jev`, the OpenAI Decisions API, Luna, Sol, Haiku, and Sonnet. This row fills when the spike reports. |

## Observations

Newest first. The session column holds a role, never a session name.

| Date | Session | Model | Task | Outcome | Evidence |
| --- | --- | --- | --- | --- | --- |
| 2026-11-13 | Sub-coordinator | Haiku | Classifying the findings and earlier blocker states in a pasted 20-line code-review text, inline with no tools | Right. All 3 findings and 4 blockers matched the reader's own, with the correct blocking flags. It invented nothing, and the fixes were short and faithful. The input was small and the reader knew the answer, so it says nothing about long logs or diffs. It wasn't the reviewer of record. | Internal record, 2026-11-13, review-text classification, right |
| 2026-11-13 | Sub-coordinator | Haiku | Classifying the rows of a tracker file that cite a PR number without a commit hash | Validated on one small run. It flagged none of the 4 rows that cite both a PR and a commit hash, matched the reader's script on every row, and put the one row with PR-like references and no commit hash under "unsure" instead of guessing. | Internal record, 2026-11-13, tracker row classification, validated |
| 2026-11-13 | Sub-coordinator | Haiku | Reading a 5,600-line CI job log for failed tests and anomalies | Right on the main points and usable for triage, but not as a reviewer of record. It found both failing test names, the assertion text, and the warnings with line citations, and 6 spot-checked lines matched. It said the log didn't show the failed workflow step instead of guessing. It also said two file hashes differed when the truncated line showed an identical prefix. It listed a summary line as a separate problem, and it flagged a follow-on error as its own. | Internal record, 2026-11-13, CI log read, right on the main points |
| 2026-11-13 | Sub-coordinator | Haiku | Mechanical check of 119 JSON run records against 5 field rules, with 3 deviations planted in a scratch copy | Validated. It reported exactly the 3 planted records with the right rule and offending value each, and counted 119 records. Every other record passed. The run took 7.2 seconds. The test used planted errors only, so it says nothing about subtle anomalies, and Sonnet stays the fallback. | Internal record, 2026-11-13, run-record check, validated |
| 2026-11-13 | Sub-coordinator | Haiku | Mechanical check of a 20-line draft file against a 7-item list of private strings | Mixed. It answered 6 of 7 items correctly. It reported one listed word as absent when a search found it once, and it flagged two harmless items as private. Use it for a first pass and keep a search or a Sonnet read as the check. | Internal record, 2026-11-13, private-string check, mixed |
| 2026-11-13 | Sub-coordinator | Haiku | Scanning a 2,470-line CI log for remote-cache failures, with one eviction line planted in a copy | Validated on one sample. It found the planted line at the right line number with exact text and flagged its out-of-sequence timestamp. On the real log it returned none found and explained each excluded keyword hit. It also reported that the log didn't mention a remote-cache flag, which a search confirmed. Both runs used search passes and said so. | Internal record, 2026-11-13, CI log scan, validated |
| 2026-11-13 | Coordinator | Haiku | Anomaly reading in a host snapshot, from inline text with no tools | Usable as a first-pass filter with a human or Sonnet check, and too noisy to act on directly. It caught the real anomaly, a load spike from 12.8 to 54.5 against the previous snapshot. It also flagged four expected or noise items, and it misread a threshold by saying a process at 23% CPU contradicted a rule that counts only processes over 50%. | Internal record, 2026-11-13, snapshot anomaly reading, mixed |
| 2026-11-13 | Coordinator | Sonnet | Read-only tracker check against the main branch, PR state, and decision files | Validated for read-only tracker verification. It checked 44 rows in 50 seconds, kept to a budget of 12 `gh` calls and used 3, and cited evidence per row. It found one stale row correctly and each "done" claim it made held up. It marked rows that need host access as uncheckable instead of guessing. | Internal record, 2026-11-13, tracker staleness check, succeeded |
| 2026-11-13 | Coordinator | Haiku | Search-and-cite | Succeeded. It traced stuck PRs and verified open items with file and line citations, and the coordinator spot-checked one. It also read a host snapshot and caught a missed flag. | Internal record, 2026-11-13, search-and-cite, succeeded |
| 2026-11-13 | Coordinator | Sonnet | Script fix | Succeeded. The fixed script passed its test with 0 failures. | Internal record, 2026-11-13, script fix, succeeded |
| 2026-11-13 | Coordinator | Sonnet | Search-and-cite | Succeeded at a 45-row tracker staleness check. The coordinator spot-checked 3 commits. | Internal record, 2026-11-13, search-and-cite, succeeded |
| 2026-11-13 | Reviewer | Codex `gpt-6.1-sol` | Code review, second family | Succeeded. It found blocking security issues in two PRs that the first-family review alone didn't state as strongly. | Internal record, 2026-11-13, second-family code review, succeeded |

## How to update

- Any session that sees a model plainly succeed or fail at a task type appends an observation, with evidence, by PR.
- Batch observations into one PR per session per day.
- The wiki's maintainer folds repeated observations into the task-fit table.
- A fit claim needs evidence or the label `unverified`.
- Write entries with [TEMPLATE.md](TEMPLATE.md), check them with [WRITING.md](WRITING.md), and review them with [REVIEW.md](REVIEW.md).

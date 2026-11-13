<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model observations

This file backs the recommendations in [README.md](README.md). Each line is one finding, grouped by model, and none is a benchmark. Every result comes from a small sample.

## Haiku

- Succeeded at classifying findings and blockers in a short code-review text, and at sorting rows of a tracker file.
- Succeeded at checking JSON run records against field rules, with planted errors only. Subtle anomalies are untested.
- Succeeded at scanning a CI log for cache failures, and said which parts it checked by search.
- Mixed on reading a long CI job log. It found the failing tests and the assertion text, but it misread two file hashes and counted one follow-on error as its own problem. Use it for triage.
- Mixed on checking a draft against a list of private strings. It missed one listed word and flagged two harmless items. Keep a search or a Sonnet read as the check.
- Mixed on reading a host snapshot. It caught a load spike, but it flagged noise and misread a CPU threshold. Treat it as a first-pass filter.
- Mixed on deciding whether a change is on main from commit subjects. It over-called "nothing yet" because work landed under different names. Give it file listings or grep output.

## Sonnet

- Succeeded at a read-only check of tracker rows against the main branch, PR state, and decision files, and marked rows that need host access as uncheckable.
- Succeeded at fixing a script, which then passed its test.

## Codex `gpt-6.1-sol`

- Succeeded at second-family code review, finding blocking security issues in two PRs that the first-family review alone stated less strongly.

## Antigravity `gemini-4-argon-high`

- Succeeded at quoting a marker line from `GEMINI.md`, from `AGENTS.md`, and from both files in one directory, in headless one-shot runs. Each file loaded without a flag.
- Succeeded at running a `PreToolUse` hook before each tool call in a session, and at reading a mid-session edit of the hooks file on the next call. The agent never rewrote the file.
- Succeeded at stopping `git push` and `git reset --hard` through a `PreToolUse` hook in headless runs. Exit code 2 and a stdout `decision` of `deny` both worked, and the model saw the reason text. A hook that crashes, times out, or lacks its interpreter is untested.
- Failed at blocking `git push` through the `--settings` permissions key. None of seven shapes tried stopped the push, and the flag rejects unknown keys such as `deny`, `hooks`, and `sandbox`. Rules saved by `/permissions` are untested.
- Mixed on settings keys. `--settings` accepted `statusLine` and `model`, but its schema isn't documented in the help text. Each key needs a probe.

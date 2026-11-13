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

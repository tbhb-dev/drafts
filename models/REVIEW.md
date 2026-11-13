<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster review checklist

The reviewer didn't write the change and reads the whole diff against [README.md](README.md), [TEMPLATE.md](TEMPLATE.md), and [WRITING.md](WRITING.md). Check each item against the cited evidence, not against the author's summary.

## Blocking: evidence

- [ ] Every new observation cites a public commit or PR, or an internal record with its date, task type, and outcome. Open at least one public source and confirm it shows the stated outcome.
- [ ] Every task-fit change rests on a standing ruling or on repeated observations in the log, and its evidence column cites them.
- [ ] Every fit claim without evidence says `unverified`.
- [ ] No row contradicts a standing ruling by @tbhb.

## Blocking: format

- [ ] Each row matches its format in TEMPLATE.md, column for column.
- [ ] New observations go at the top, and the table remains in newest-first order.
- [ ] Every model in a task-fit row or observation has a roster row.

## Blocking: privacy and spending

- [ ] The change has no secrets, public IP addresses, host names, tailnet details, internal paths, private repository names, PR numbers from private repositories, account addresses, or session names.
- [ ] A cost cell with API spend cites @tbhb's approval and its budget. The change never adds spend without one.

## Non-blocking: writing

- [ ] Outcomes are one or two plain sentences.
- [ ] The same model and task type keep the same name everywhere.

## Report

Report as a PR review, in this format:

```text
Change: <owner/repo#N> at <sha>
Verdict: ready to merge | changes needed
Blocking:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
Non-blocking:
- <file:line>: <note>
Checked: <the sources opened, and what each showed>
```

The verdict covers only that `sha`. A new commit needs a fresh review of at least the diff.

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- The placeholders in angle brackets are template text. -->
<!-- rumdl-disable MD033 -->

# Model roster entry template

Copy the row that matches the change into [README.md](README.md). Replace each placeholder in angle brackets and keep the column order.

## Observation

Add the row at the top of the Observations table, under the header row.

```markdown
| <yyyy-mm-dd> | <role, never a session name> | <model as the roster names it> | <task type from the task-fit table> | <Succeeded or Failed. One or two sentences on what it did.> | <public commit or PR as owner/repo#N, or "Internal record, yyyy-mm-dd, task type, outcome"> |
```

## Task-fit row

Change a task-fit row only when observations or a ruling support it.

```markdown
| <task type> | <model, and effort if a ruling sets one> | <model or Unverified> | <model or Unverified> | <ruling with its internal record date, or observation dates and models> |
```

## Roster row

Add a roster row for a model before any task-fit row or observation cites it.

```markdown
| <model> | <vendor>, <harness> | <account or billing, without addresses> | <Subscription quota or API spend with its approval> | <known limits, or Unverified> |
```

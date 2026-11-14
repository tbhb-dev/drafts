<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Off-host copy of the administrator audit log

| Status | Owner | Updated | Open decisions | Needs from the operator |
| --- | --- | --- | --- | --- |
| Draft | Platform engineering | 2026-11-14 | 3 | Nothing yet |

Draft of 2026-11-14. It proposes a copy mechanism that `tbhbdev admin` doesn't have today. The operator decided on 2026-11-14 that an off-host copy of the audit log needs the receiver's acknowledgment before the tool's first credential read, with the tool's user as the trusted boundary on its host.

## Summary

The tool appends each call to a local, hash-chained log. A process running as the same user can still rewrite that file. This draft adds a receiver on another host. The tool sends every new line there and reads the App key only after the receiver acknowledges the line.

## Proposal

- **The receiver.** A new unprivileged user on a lab host that Coordination picks after Ops answers. The user has no shell. Its only authorized key has the `restrict` option and a forced command that appends one line to a file in the user's home. Ops creates the user, the file, and the key entry. Nobody else on the receiver can write the file.
- **The forced command.** It reads one line from standard input and requires valid JSON. It requires `prev` to equal the Secure Hash Algorithm (SHA) 256 digest of the file's current last line, or 64 zeros for an empty file. It appends the line with `O_APPEND` and syncs the file. Then it prints `ack 1 <n> <digest>`, where `n` is the line count and `digest` is the SHA digest of the appended line. A line equal to the current last line gets the same acknowledgment without a second append, so a retry is safe. Any other line fails with a nonzero exit and no acknowledgment.
- **The sender.** After each append to `calls.jsonl`, the tool runs `ssh -o BatchMode=yes` to the receiver and sends each line the receiver hasn't acknowledged, oldest first. The tool records the last acknowledged digest in a state file beside the log.
- **The gate.** For a call that reads the key, the tool appends the `before` record and sends the backlog with that record. It then waits for an acknowledgment whose digest matches the record. It refuses with `audit_copy_unacknowledged` when the wait ends without one. The key read and the HTTP request come after the gate and never before it.
- **Calls without a key read.** Dry runs and policy refusals write the local record and try to send it. A failure doesn't stop them. The receiver's chain check then forces the next key call to send those lines first.

## Failure behavior

The tool refuses and never skips. An unreachable receiver, a timeout (10 seconds, open question D2), a rejected line, or a digest mismatch ends the key call before the key read without falling back to the local log alone. An `after` record that fails to send stays in the backlog, and the next key call sends it before anything else. The audit record of a refused call says why.

## Dependence of the `enabled` marker

The marker file only opts a host in. After this change the key read also requires a receiver setting in the audit directory, that gives the receiver user and host, and a successful gate. Ops creates the marker last, after the receiver accepts a test line from the key host. A host with the marker and no receiver setting refuses with `credential_unavailable`.

## Guarantee and limits

The receiver holds acknowledged history. A reader can compare it with the local log and find a deleted or edited line, because each line contains the digest of the one before it. A compromised tool user can still skip the tool and act without a record, as the operator's decision accepts. The ssh key the tool uses for the receiver is a credential and belongs in the key host's own user account, never in a shared one.

## Open questions

- **D1.** Which record the gate waits on: the `before` record just written (this draft), or the previous record's digest as Coordination first described. This draft chose the stronger one because the acknowledgment then covers the call's own intent. Undecided.
- **D2.** The timeout length and whether the tool retries inside it. Undecided.
- **D3.** Receiver file rotation and size limits. The file grows by one line per call, so rotation can wait, though it needs a rule before the file gets large. Undecided.

## Plan

1. Review this draft (one family, docs only).
2. Ops creates the receiver user, file and key entry on the host Coordination picks.
3. A tools pull request adds the sender, the gate and the receiver command, with tests against a fake transport. It touches the credential path, so both reviewer families review it.
4. Ops tests a line from the key host, creates the marker, and the ruleset request waits on the Coordination go.

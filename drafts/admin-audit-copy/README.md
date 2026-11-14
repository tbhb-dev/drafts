<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Off-host copy of the administrator audit log

| Status | Owner | Updated | Open decisions | Needs from the operator |
| --- | --- | --- | --- | --- |
| Draft | Platform engineering | 2026-11-14 | 0 | Nothing |

Draft of 2026-11-14, proposing a copy mechanism that `tbhbdev admin` doesn't have today. The hash chain and the `enabled` marker it builds on exist only after tools#197 merges, since neither is on `main` yet.

On 2026-11-14 the operator decided (record `2026-11-14-open-233`) that an off-host copy of the audit log needs the receiver's acknowledgment before the tool's first credential read, with the tool's user as the trusted boundary on its host. That answer came on the earlier broker design draft. Nobody built the broker, because the administrator VM's own tool replaced it. This draft applies the answer to that tool.

## Summary

The tool appends each call to a local, hash-chained log. A process running as the same user can still rewrite that file. This draft adds a receiver on another host. The tool sends every new line there and reads the App key only after the receiver acknowledges the line.

## Proposal

- **The receiver.** An unprivileged account on a Linux lab host. Coordination picks the host, and the decision record `2026-11-14-audit-receiver-linux-lab-host` holds that choice. Ops creates the account, the log file in its home, and one `authorized_keys` entry. The entry uses the `restrict` option, which turns off terminals and forwarding, and a forced command. Nobody uses the account interactively. Its login shell runs only the forced command. The host needs a plain shell that follows the Portable Operating System Interface (POSIX) standard.
- **The forced command.** It takes two requests. An append request reads one line from standard input and requires valid JSON under 64 KiB. It requires `prev` to equal the Secure Hash Algorithm (SHA) 256 digest of the file's current last line, or 64 zeros for an empty file. It appends the line with `O_APPEND` and syncs the file. Then it prints `ack 1 <n> <digest>`, where `n` is the line count and `digest` is the SHA digest of the appended line. A line equal to the current last line gets the same acknowledgment without a second append, so a retry is safe. Any other line fails with a nonzero exit and no acknowledgment. A head request is read-only and prints the same `ack 1 <n> <digest>` for the current last line.
- **The sender.** After each append to `calls.jsonl`, the tool runs `ssh` with `BatchMode=yes` and `StrictHostKeyChecking=yes` against a pinned `known_hosts` entry in the audit directory. It first sends a head request and finds the acknowledged digest in its local log. Then it sends each later line, oldest first. If the digest is absent from the local log, the tool refuses. A state file beside the log caches the last acknowledgment but anchors nothing. After every acknowledgment the tool checks `n` against its own line count and the digest against the line it sent.
- **The gate.** Before a call reads the key, the tool appends the `before` record and sends the backlog with it. It then waits for an acknowledgment whose digest matches the record, and without one it refuses with `audit_copy_unacknowledged`. The key read and the HTTP request come after that check. The `after` record isn't gated. The call has already run by then, so refusing would only drop a record, and the next key call sends the backlog first.
- **Calls without a key read.** Dry runs and policy refusals write the local record and try to send it. A failure doesn't stop them. The receiver's chain check then forces the next key call to send those lines first.

## Failure behavior

Refusal replaces skipping. The key call ends before the key read when the receiver is unreachable, when a send times out, when the receiver rejects a line, or when a digest differs. It never falls back to the local log alone. `ConnectTimeout` and a total limit of 10 seconds bound each send. The tool doesn't retry inside that window, though the receiver accepts one identical last line twice. A refused call's audit record says why.

A second sender or an overlapping send meets the receiver's chain check and fails there. That rejection ends the call as a refusal, which fails closed and costs the caller a retry. A send takes no lock.

## Dependence of the `enabled` marker

The marker file only opts a host in. After this change the key read also needs a receiver setting in the audit directory, which gives the receiver account and host, and a passing gate. The administrator VM session creates the marker last, after the receiver accepts a test line from the key host. A host with the marker and no receiver setting refuses with `credential_unavailable`.

## Guarantee and limits

The receiver keeps acknowledged history. A reader can compare it with the local log and find an edited earlier line, because each line contains the digest of the one before it. A compromised tool user can skip the tool and act without a record, as the operator's decision accepts. The same user can also send a forged line that breaks the chain's agreement with the receiver. That attack only denies service, because the receiver then rejects later lines and the gate refuses. The ssh key for the receiver is a credential and belongs in the key host's own user account.

## Decisions

Coordination decided these on 2026-11-14 as two-way doors. Each stays reversible in a later change.

- **D1.** The gate waits on the acknowledgment of the `before` record just written, at the cost of one ssh round trip per key call. Waiting on the previous record's digest would leave the call's own intent uncovered.
- **D2.** `ConnectTimeout` plus a total limit of 10 seconds. The window permits one retry only, the identical last line the receiver accepts.
- **D3.** Rotation by size. The receiver rejects a line over 64 KiB, which also stops a compromised sender from filling the volume. Ops picks the rotation threshold.
- **Receiver.** An unprivileged account on a Linux lab host.

## Plan

1. Review this draft.
2. On the host Coordination picks, Ops creates the receiver account, the log file, and the key entry.
3. A tools pull request adds the sender, the gate and the receiver command, with tests against a fake transport. It touches the credential path, so both reviewer families review it.
4. Ops tests a line from the key host. Then the administrator VM session creates the marker. After that, only Coordination's go stands between the ruleset request and the call.

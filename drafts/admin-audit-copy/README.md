<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Off-host copy of the administrator audit log

| Status | Owner | Updated | Open decisions | Needs from the operator |
| --- | --- | --- | --- | --- |
| Draft | Platform engineering | 2026-11-14 | 0 | Review |

This draft proposes a copy mechanism that `tbhbdev admin` doesn't have yet. It builds on the hash chain in `audit.py`, which tbhb-dev/tools#197 merged into tools `main` (read). A report, unverified here, says the operator accepted one principle for this copy. The receiver must acknowledge a call before the tool's first credential read, and the tool's user marks the trusted boundary on its own host. This draft applies that principle.

## Summary

The tool appends each call to a local, hash-chained log. Any process running as the same user can rewrite that file. This draft adds a receiver on another host. The tool sends every new line there, and it reads the App key only after the receiver acknowledges the call's own `before` record. When the copy fails, the call refuses instead of skipping the copy.

## The local log today

Everything in this section comes from reading `audit.py` and `request.py` on tools `main`.

- `Audit.write` opens `calls.jsonl` with `O_APPEND` and takes an exclusive `flock`. Under that lock it confirms the file is still the log and runs `heal_tail`. It computes `prev` with `chain_head` and writes one JSON line. A sync of the file follows. When the write or the sync fails, `roll_back` truncates the file to its earlier size and syncs it.
- `chain_head` returns the Secure Hash Algorithm (SHA) 256 hex digest of the last line, without its trailing line feed (LF). It reads backward in 4096-byte blocks. An empty file yields 64 zeros.
- `heal_tail` appends one LF when the file doesn't end in one. It leaves the unfinished bytes in place. Its docstring gives a failed `roll_back` as the cause. A writer that stopped in the middle of a write would leave the same kind of tail (inferred, not tested). The fragment then becomes a line of its own, and the next record's `prev` hashes it. A fragment parses as JSON only when the writer lost nothing but the LF.
- `json.dumps` runs with its default arguments and escapes every character outside bytes 32 through 126. Each line contains only those bytes.
- `write` doesn't limit the size of a record. The caller supplies `reason` and `path`.

## Proposal

### Receiver

An unprivileged account on a Linux host in the lab receives the copy, and the receiver's operator picks the host. Its forced command runs a receiver module. According to `pyproject.toml`, the `tbhbdev` package requires Python 3.15 or later and depends on seven third-party distributions, among them `cryptography` and `httpx`, and the receiver needs none of them. In the mechanism pull request, the receiver comes as one standalone module that imports only the standard library. The receiver's operator installs Python 3.15 or later on the receiver host and copies that file into the account. The account's directory contains numbered log segments, a lock file, and a head file. In the head file, one line of the form `<segment> <n> <digest> <length>` records the receiver's position, where `n` counts every line the receiver has committed across all segments, including lines whose acknowledgment the sender never received, `digest` is the SHA-256 of the last such line, and `length` gives the byte size of the current segment up to that line. A fresh receiver starts at `1 0 <64 zeros> 0`. Segment 1 starts empty.

Each request runs under an exclusive lock on the lock file. The forced command keeps that lock from recovery through validation, append, sync, and head update to the acknowledgment. Senders never interleave, and each request sees the head the previous one left.

### Wire contract

The forced command reads the request from `SSH_ORIGINAL_COMMAND`. It accepts exactly two forms and exits nonzero with empty standard output for anything else.

- `head` takes no input. The receiver reads standard input to end of file, and any byte there ends the request with a nonzero exit and empty standard output. Otherwise it prints the acknowledgment for the current head.
- `append K`, where `K` is a decimal integer of at least 1 without leading zeros, names the position of the line in the sender's log. Standard input contains exactly one line of at most 65536 bytes, LF included, followed by end of file. The receiver reads at most 65537 bytes and rejects longer input.

For `append K`, the receiver compares `K` with its count `n`.

- When `K` is at most `n`, the receiver doesn't append and prints the acknowledgment for its current head. The reply doesn't say whether the sent line equals the receiver's line `K`. The sender's digest check compares only local line `n` with the receiver's line `n`. Before it replies, the receiver still reads standard input to end of file, up to 65537 bytes. It checks that input against the size limit and the range of allowed bytes. It skips the JSON and `prev` checks. Input that fails ends the request with a nonzero exit and empty standard output. The sender writes the line and closes standard input. When the receiver exits early, after input over the size limit or with a disallowed byte, or when its recovery step ends the request, the broken pipe counts as a failed request, handled like any other failure.
- When `K` equals `n + 1`, the receiver validates the line. Every byte before the final LF must have a value from 32 through 126. Those bytes must parse as one JSON object under Request for Comments (RFC) 8259, without `NaN` or `Infinity` and without a duplicate member name at any depth. A `prev` member must exist, and its value must be a string of 64 lowercase hex digits equal to the head digest. The receiver doesn't check other members. A valid line goes to the end of the current segment.
- A larger `K`, or a line that fails validation, ends the request with a nonzero exit and empty standard output.

Every digest in this contract is the SHA-256 of the line's raw bytes without the trailing LF, the same input `chain_head` hashes.

An acknowledgment consists of `ack 1 <n> <digest>` and one LF, with exit status 0. Its `1` names the protocol version. `n` has no leading zeros, and `digest` uses 64 lowercase hex digits. The sender reads at most 128 bytes of standard output. Any other byte, a missing LF, or a nonzero exit counts as a failure, whatever standard error contains.

### Sender

At install, the receiver's operator writes the receiver setting, with the account and host, into the audit directory on the key host. An ssh configuration file and a `known_hosts` file with the receiver's pinned host key go beside it. The tool runs `ssh -F` with that configuration. On the command line it also passes `BatchMode=yes` and `StrictHostKeyChecking=yes`. It sets `GlobalKnownHostsFile=/dev/null`, `UserKnownHostsFile` to the pinned file, and `VerifyHostKeyDNS=no` there too. Options on the command line take precedence over the file. The pinned file contains no `@cert-authority` lines. With those settings ssh accepts only the pinned host key. At install, the receiver's operator reads the effective values with `ssh -G` and records them.

After each local append, `write` returns the position `K` of the new line, counted under the lock. That return value is new code in `audit.py`. To bring the receiver's count up to `K`, the sender starts from the count `n` cached in a state file beside the log, along with the digest of local line `n`. Before it uses the cache, the sender checks that local line `n` exists and hashes to the cached digest. A missing line or a different digest clears the cache. With no cache it sends `head` first. For a shortened or replaced log, that reply fails the check below, and the call refuses with `audit_copy_diverged`.

The sender reads and writes the state file and sends only under an exclusive lock on a lock file beside it. It sends each later line in order, one `append` request per line, oldest first.

Every acknowledgment `ack 1 <n> <digest>` passes one check. With `n` equal to 0, the digest must equal 64 zeros. Otherwise local line `n` must exist, and its digest must equal `digest`. A match shows only that local line `n` equals the receiver's line `n`. It doesn't show that the earlier lines match: after an edit to local line 1 that leaves line 2 alone, the acknowledgment for line 2 still passes. After a passing check the sender caches `n` and the digest and continues from `n + 1`. A failing check refuses the call with `audit_copy_diverged`. For an empty receiver, `head` reports `ack 1 0 <64 zeros>`, and sending starts at local line 1. In any log that `audit.py` started, line 1 has a `prev` of 64 zeros.

Before it sends a line, the sender applies the receiver's validation itself. When a line fails, the sender first sends `head`, in case a repair already moved the receiver past that line, and refuses the call with `audit_copy_damaged` if the receiver's count is still below the line's position. The sender also checks that each line's `prev` equals the digest of the previous local line, which it reads anyway, and refuses with `audit_copy_diverged` on a mismatch. That check finds edits inside the range it sends. Lines at or below the receiver's count aren't checked again. An earlier local edit that the sender never reads again comes to light only when a reader compares the two logs.

### Gate

Before a call reads the key, the tool appends the `before` record and sends the backlog through that record. It reads the key only after an acknowledgment with `n` at least the record's position passes the check. Without one it refuses with `audit_copy_unacknowledged`. Only then come the key read and the request to GitHub. After the `after` record, the tool tries one more send, but the call's result doesn't depend on it. By then the call has already acted, and the next key call sends any unsent line first.

Dry runs and policy refusals also append and try to send. A failed send doesn't stop them. Those lines then go out ahead of the next key call's own line.

The `enabled` marker keeps its meaning and opts a host in. On top of it, the key read needs the receiver setting and a passing gate. A host with the marker and without the receiver setting refuses with `credential_unavailable`. The operator of the key host creates the marker last, after the install test passes.

## Failure and crash behavior

A call copies in two phases. The gated phase runs from the `before` record to its acknowledgment, and the trailing phase sends the `after` record. Each phase gets one deadline of 10 seconds across all its ssh connections, and each connection also uses `ConnectTimeout`. The sender makes one attempt per request and doesn't retry within a call. In the gated phase, a timeout, a lost connection, a rejection, or a malformed acknowledgment refuses the call with `audit_copy_unacknowledged`, and the refused call's local record names the code. A trailing failure leaves the call's result alone. After a rejected `append`, the sender also clears its cached count, and the next call starts with `head`. Retrying in a later call is safe, because a repeated `K` gets the receiver's head in reply. Each passing acknowledgment updates the cached count and digest. A backlog too large for one deadline shrinks across successive refused calls only when each call sends more lines than it adds. Every refused call appends its `before` record and a refusal record, and dry runs append records of their own. Consider a link so slow that a call sends two lines or fewer before its deadline. There the backlog never shrinks through key calls. For that case the mechanism pull request adds a catch-up subcommand to the tool. It sends the backlog by the same per-request rules without the 10-second deadline. It takes the sender's lock, the same one key calls use, and updates or clears the cached count and digest by the same rules. It doesn't write a call record, and it never reads the key. Running it drains the backlog.

With a cached count and no backlog, a key call makes two connections, one per phase. Without a cache the sender adds a `head` request, and each backlog line adds one more. An environment write makes at least four, because its lookup of the current environment is a gated call of its own.

An acknowledgment covers only durable history. Each request runs these steps under the lock, and the head file records what that history includes.

1. Truncate the current segment to the head file's `length` and sync it. A partial write from an earlier request goes away here, along with any full line whose head update never happened. This step also removes any leftover temporary head file and any segment file with a number higher than the head's segment.
2. Check the request.
3. Rotate when the line would push the current segment past the segment size. The receiver creates the next segment as an empty file, truncating any leftover file of that name, and syncs the directory. Then it commits a head file that points at the new segment with `length` 0, the same `n`, and the same digest, by the method in step 5.
4. Append the line with `O_APPEND` and sync the segment.
5. Write a new head file to a temporary name, sync it, rename it over the old one, and sync the directory.
6. Sync the head file and the directory, then print the acknowledgment. When either sync fails, the request exits nonzero with empty standard output.

Step 6 ends every request that replies with an acknowledgment. An append runs it after step 5, and that sync covers the directory entry of the head file step 5 renamed. A `head` request and an `append K` with `K` at most `n` skip steps 3 through 5 and run step 6 right after step 2. When a rename made a head file visible but the directory sync failed, a later request acknowledges that head only after both syncs succeed. After a crash the directory shows the old head file, the rotation head from step 3, or the new head from step 5. Each is consistent. Step 4 syncs the line before step 5 replaces the head file, and step 1 truncates the segment to the visible head's `length`. A rotation head points at the new segment, which is empty, with `length` 0. A crash inside step 3 before its head commit leaves at most an empty segment that the head file doesn't name, and the next request's step 1 removes it. A failed sync ends the request without an acknowledgment, and step 1 runs again next time. After step 5, a crash leaves a durable line without an acknowledgment. The sender refuses that call, and its next request reads the new count from the reply. The receiver rejects every request while a segment is shorter than `length` or after a failed truncation, until its operator repairs that state.

When the sender detects a disagreement between the receiver's history and the local log, it refuses every key call. The sender's checks don't catch an edit to an older local line when the sender doesn't read that line again and the lines it does read still link through `prev`. The receiver's operator repairs the receiver chain after the operator decides how.

A damaged local line, such as a fragment that `heal_tail` ended, blocks every key call with `audit_copy_damaged`. Nobody can fix the fragment in place, because the next record's `prev` hashes it. Recovery needs the operator. After comparing the bytes with the key host's log, the receiver's operator appends that one line on the receiver host. A local repair command does this, skipping the JSON check but keeping the size limit and the byte range check. Because the receiver doesn't check that line's `prev`, the operator's record of the byte comparison is what readers rely on for it. Once the receiver's count passes the damaged line, sending resumes. During the install test a damaged line has a simpler fix. The operator of the key host moves the old log aside without deleting it and removes the sender's cached count, the state file beside the log. The receiver's operator moves the receiver's segments, head file, and lock file aside in the same way, then writes a fresh head file, `1 0 <64 zeros> 0`, and an empty segment 1. Only then does the test run again against a new log. A new log alone fails the digest check with `audit_copy_diverged`, because the receiver already has the lines before the damaged one.

## Trust boundary

All files of the receiver belong to the receiver account. Its directory has mode 0700, and the segments, the head file, and the lock file have mode 0600. The account has one `authorized_keys` entry, which sets the `restrict` option and the forced command, and no other entry. The `restrict` option turns off terminal allocation and forwarding. It also skips the user's ssh startup file. The receiver's operator locks the account's password, and for that user the ssh server's configuration turns off password and keyboard-interactive login. Those settings alone don't exclude other ways in, such as site-wide certificate authority trust, another authorized-key provider, or client-sent environment that reaches the shell or Python before the forced command. At install the receiver's operator reads the effective server configuration for that user, such as the output of `sshd -T -C user=<account>`. That reading covers `AuthorizedKeysFile`, `AuthorizedKeysCommand`, `TrustedUserCAKeys`, `PermitUserEnvironment`, `AcceptEnv`, and every authorized-key source for the account, and the operator records in the install record that this reading is how they checked the claim. Only then does this draft treat that one entry as the key holder's way into the account. The mechanism pull request requires the forced command to start Python in isolated mode with `-I`. In that mode the interpreter ignores its environment variables and leaves the working directory off the import path. This draft assumes that `sshd` starts the forced command through the account's login shell, which would then need to be a normal shell that follows the Portable Operating System Interface (POSIX). Nobody has read or tested that here, and the mechanism pull request tests it on the receiver host. The forced command ignores everything the client sends except `SSH_ORIGINAL_COMMAND`, which it parses as one of its two verbs, and the one standard input line the wire contract describes.

Inside that boundary, a key holder can't edit committed lines. Outside it are anyone who controls the receiver account and anyone with root on the receiver host. Either can rewrite history, and this draft treats both as out of scope.

## Rotation and disk bound

The receiver's operator sets a segment size and a total quota. The segment size must be at least 65536 bytes. Any accepted line then fits in an empty segment. The quota must be at least two segment sizes. The operator can then move a full old segment off the host while the current one fills. Before an append, the receiver checks whether the line would push the current segment past the segment size. In that case it starts the next segment through step 3 of the crash list, and the head file gets the new segment number with `length` 0. Both the count `n` and the digest continue unchanged. In the new segment, the first line has a `prev` equal to the last digest of the old one. A reader checks the chain across segments by reading them in order. A line the repair command added is the exception, even as the first line of a segment. It may not parse as JSON, and the receiver didn't check its `prev`. For that line the reader relies on the operator's record of the byte comparison. For `head`, the receiver reads the head file and runs steps 1 and 6 of the crash list under the lock. That step may truncate the current segment, but `head` doesn't read any segment line.

Rotation never deletes a segment. Instead the receiver rejects an append that would push the sum of its segments past the quota. Disk use can pass the quota only by a bounded margin. That margin covers the head file, the lock file, the temporary head file, and one partial record that the next request's step 1 truncates. Key calls refuse once the quota fills. The receiver's operator then moves old segments off the host, after the operator decides where they go.

Local records need a matching limit. Without one, a record over 65536 bytes, LF included, would block every later key call. In the mechanism pull request, `audit.py` gains a check that refuses any line over 65536 bytes, LF included, before the local append, with the code `audit_record_too_large`. A call checks its `before` record against 1024 bytes less than that and leaves room for the `after` record. The same pull request bounds the `after` record at 1024 bytes, LF included. By then the call has acted, and `audit.py` shortens the free-text fields of a larger `after` record, such as an echoed `reason` or `path`, until the record fits. As written, this draft doesn't change tools code.

## Decisions

All three are two-way doors. A report says another party decided them. Lacking a public record of that decision, this draft labels each one proposed.

- **D1, proposed.** The gate waits on the acknowledgment of the call's own `before` record. A gate on the previous record would leave the call's intent outside the copy. The price is at least two ssh connections per key call. One runs before the key read and one after the call.
- **D2, proposed.** `ConnectTimeout` plus one deadline of 10 seconds for each copy phase, across all its connections. A call doesn't retry. An earlier version allowed one retry of an identical last line, and this one drops it in favor of resending in a later call.
- **D3, proposed.** Rotation by segment size, with a per-line limit of 65536 bytes, LF included, and a total quota on the receiver. Rotation keeps the count and the chain going across segments.

## Limits

- A process running as the tool's user can forge a record with a correct `prev` and send it after the local append. The receiver accepts it.
- The same process can rewrite any local line the receiver hasn't committed yet.
- The digest check proves just that local line `n` equals the receiver's line `n`. When the sender won't read an older local line again, and the lines it does read still link through `prev`, none of the sender's checks catch an edit to that line. Such an edit shows up only when a reader compares the two logs.
- It can also skip the tool and act without a record. A report, unverified here, says the operator accepted this limit.
- A sender can fill the quota or send lines the receiver rejects. Either only denies service, because key calls then refuse.
- The copy proves nothing until a reader compares it with the local log.
- Positions assume one local log that only grows. Rotating, shortening, or replacing `calls.jsonl` fails the cache check and then the digest check of the `head` reply, and key calls refuse with `audit_copy_diverged` until the operator decides how to continue.
- The receiver's ssh key sits in the tool user's account on the key host, where that user can read it.

## Plan

1. Review this draft.
2. A tools pull request adds the sender, the gate, the catch-up subcommand, the receiver module, the repair command, the position return value, and the record limit in `audit.py`, with tests against a fake transport. It touches the credential path and needs two independent reviews.
3. On the receiver host, its operator creates the account, the directory, the fresh head file, the ssh server settings, and the key entry. That operator then writes the receiver setting and the ssh configuration on the key host, along with the pinned `known_hosts` file.
4. For the install test, someone first logs in over ssh as the key holder, with the effective restrictions in place, and runs `head`. It must return the fresh receiver's acknowledgment. Next, someone runs a dry run on the key host. That writes its `before` and `after` records, and the sender sends local lines from line 1 through the `after` record. The test passes when the receiver's head matches the local head. Only with a new local log and a fresh receiver does the dry run's `before` record become the receiver's first line.
5. The operator of the key host creates the `enabled` marker.

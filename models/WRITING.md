<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster writer checklist

Follow this list before opening a PR against [README.md](README.md) or [OBSERVATIONS.md](OBSERVATIONS.md). Use the formats in [TEMPLATE.md](TEMPLATE.md).

## Page shape

- [ ] The README has standing rulings, the Models table, and what's undecided. It has no dates, run ids, session names, spot-checker names, sample counts, or timestamps.
- [ ] Evidence goes in OBSERVATIONS.md as one line per finding. A mixed result is one line that says Mixed and what to do about it.
- [ ] A recommendation rests on a ruling or an observation. A claim without support says `unverified`.
- [ ] A row never contradicts a standing ruling.
- [ ] A cost statement says subscription quota or API spend, and API spend cites @tbhb's approval.

## Privacy and style

- [ ] The change has no secrets, public IP addresses, host names, tailnet details, internal paths, private repository names, account addresses, or session names.
- [ ] Each Markdown paragraph is on one line.
- [ ] `vale`, `rumdl check`, and `gitleaks dir . --no-banner` pass on the changed files, and the outgoing private-name scan passes before the push.

## The PR

- [ ] The PR body fills every section of the PR template.

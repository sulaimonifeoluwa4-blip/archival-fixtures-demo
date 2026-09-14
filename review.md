# Review — `archival-fixtures-demo` (updated 2026-09-14: closeout pass)

A self-review of the repo as it stands after the verification/hardening
pass: the earlier review marked everything "verified by construction"; this
one records what is now **verified by execution** against the live network,
what was found and fixed, and what is still pending or blocked.

## Closeout pass (2026-09-14) — issue #2 automation, and three stale records

This pass set out to close issues #3–#7 and land PR #9. Reconnaissance found
that had already happened: **#3, #4, #5, #6 and #7 are closed, and PRs #9
(`docs/submission-drafts`, `6515312`) and #10 (`feat/standalone-rehearsal`,
`89dd098`) are merged** — local `main` matched `upstream/main` exactly. Issue
#2 was the only open item, so the pass became a verification of the closed
items plus one real fix.

### Fixed: issue #2's automation could never self-close

`capture-transcript.yml` (added in #10) dispatches `demo-restore.yml` when the
entry reaches the Archived band, but **nothing anywhere wrote
`.transcripts/07-restore.txt`** — while that same workflow's final step gated
its comment on issue #2 on exactly that file existing. The "all transcripts
present" condition was unreachable by construction, so #2 could not close
itself however long anyone waited.

- `585ebb6` — new `capture-restore-transcript.yml`. On a **successful**
  `demo-restore` run it fetches the run's real log (`gh run view --log`) into
  `.transcripts/07-restore.txt`, adds the
  `.transcripts/08-scan-post-restore.json` that `docs/demo-recording-script.md`
  already expected, and opens one PR on the fixed branch
  `auto/transcripts-restore` (a re-run updates it instead of stacking new
  PRs). Failed runs are never captured, so an unset
  `TESTNET_THROWAWAY_SECRET_KEY` (issue #5) leaves it green and silent rather
  than committing a broken run as demo evidence. `demo-restore.yml` is
  deliberately untouched: its execution path cannot be exercised from CI until
  that secret exists, so the finished run's log is fetched rather than having
  the workflow report on itself.
- `e361f58` — removed a dead duplicate TTL assignment in
  `capture-transcript.yml`: the first call (stderr merged in, `tail -1`) was
  discarded by the second, and its comment described the opposite convention.
  Verified live that `read-entry-ttl.py` prints the integer as the sole stdout
  line, so a single plain capture is both correct and consistent with
  `demo-scan.yml`.

**Verification performed:** all five workflow YAMLs parse; every `run:` block
passes `bash -n` (with `${{ }}` expressions neutralized, since those are not
bash); the TTL parse fix was confirmed against the live testnet RPC (stdout is
a bare integer, `32539` at capture); and the new JSON writer plus the "still
Archived after restore" guard were exercised locally.

### Three closed items whose records are wrong or stale

1. **Issue #5's closing evidence is false.** It cites run `34832138820` as
   proof that `demo-scan.yml` is "green and running on schedule". That run was
   a manual `workflow_dispatch` and is the **only** green run in the repo's
   history — **all 22 scheduled runs have failed**, every one back to
   2026-09-08, with `CONTRACT_ID repository variable is not set`. The pattern
   is consistent with the variable simply having been set on 09-14 between
   02:28 and 10:14 (scheduled and dispatched runs read the same `vars`
   context, so there is no event-specific cause), but the schedule has not
   succeeded once. The 12:00 UTC cron is its first real test. The secret named
   in #5 is likewise still unset and still blocks `demo-restore.yml`.
2. **Issue #4's blocker has cleared.** It was closed because
   `action-state-watch` was not public. That repo is now **public**, with a
   readable `.github/workflows/self-check.yml`, so the consumed schema can now
   be read from the consumer instead of invented. Closed is not the same as
   done here.
3. **Issue #7's redirect never happened.** Its closing comment said an
   equivalent issue "should be opened" in the sentinel repo.
   `Aycode01/soroban-state-sentinel` is public, has issues enabled, and has
   **zero issues**. The need was recorded as redirected without being
   redirected, so it is currently tracked nowhere.

### Still needs a human, not a workflow

Transcript PRs still require a merge under `enforce_admins: true` plus the
required `contract-tests` check, so the wait no longer needs someone
*watching*, but it does need someone to merge — unless auto-merge is enabled
on the repo, which this environment cannot verify. Entry state at capture:
**TTL 32,539 ledgers ≈ 45 h** (latest ledger 4,672,085), Healthy. Critical
lands ~Sep 15, Archived ~Sep 16.

## Purpose (what this repo is for)

A fixture/demo suite that makes Soroban **state archival** observable
end-to-end on testnet: one deliberately short-lived persistent entry,
created at the network-minimum TTL and never extended, so it decays
Healthy → Critical → Archived on a real-time timeline and must be restored.
It exists to give the sibling tools — `soroban-state-sentinel` (scan +
unsigned remediation XDR) and `action-state-watch` (the Action wrapper) —
a real, decaying testnet entry to watch, not a mock.

## Review method (updated)

| Area | Verified against |
|---|---|
| Network TTL/rent constants | Live testnet RPC `getLedgerEntries` (protocol 28, latest ledger 4,583,387+, 2026-09-09) — decoded `STATE_ARCHIVAL` and `CONTRACT_LEDGER_COST_V0` XDR by hand |
| Sentinel CLI/JSON contract | The sentinel's source (`args.rs`, `SCHEMA.md` 1.1.0) **and the real binary built from source (0.1.0)** |
| stellar CLI flags | The actually-installed CLI 28.0.0 (`--help` for `restore`/`read`/`extend`/`keys generate`) |
| Scripts | Executed end-to-end against testnet (deploy, sentinel scan, watcher, off-chain read) |
| Contract | `cargo test` (5/5) + release WASM build, both also green in GitHub Actions |
| CI | Live GitHub Actions runs (test-contract passed; demo-scan scheduled runs recorded) |
| Repo hygiene | Full `find` of `.git`/`.gitkeep`/`.gitignore`, `git status --ignored`, `git check-ignore` |
| README banner convention | Live GitHub: `karagozemin/Sub-Rosa` (top-level `assets/` dir + `<p align="center">` wrapper); `soroban-state-sentinel`, `carbonledger`, `back-it-onchain` have no banner to contradict it |

## Verdict by area (updated)

### Contract (`contracts/rapid-expiry-demo/`) — solid, and now compiles

- Small, single-purpose: one persistent entry `VALUE`, functions
  `initialize` / `read` / `touch` / `extend`. No hidden state.
- **`ttl()` was removed** (commit `9eed1a5`): the completion pass found the
  contract did not compile — soroban-sdk 27.0.6 has no production TTL
  getter (`get_ttl` is testutils-only), per CAP-0046-12's design that
  contracts cannot read their own TTL. `extend` no longer returns the TTL,
  and all TTL reads moved off-chain. This is the production pattern the
  repo teaches, not a workaround.
- Constants (`MIN_PERSISTENT_TTL_LEDGERS` 120,960, `MIN_TEMP_TTL_LEDGERS`
  720, `MAX_ENTRY_TTL_LEDGERS` 3,110,400) **match the live ledger** —
  verified 2026-09-09.
- Unit tests configure the test ledger with the real testnet parameters,
  assert TTL via the testutils trait, and disable SDK 27's test-snapshot
  files. **`cargo test`: 5/5 passing**, and the release WASM builds
  (`wasm32v1-none` target — soroban-sdk 27 no longer supports the legacy
  `wasm32-unknown-unknown`).

### Scripts (`scripts/`) — execute correctly against the real toolchain

- The schema fix from the earlier session (`5da9d50`) was confirmed against
  the **real sentinel binary**, which surfaced one more real mismatch: the
  `--keys` SCVal must be XDR 4-byte-aligned — `Symbol "VALUE"` needs 3
  padding bytes or stellar-xdr rejects it (`27b49fb`).
- `stellar keys generate --as-secret` (CLI 28) does not print the secret —
  it stores it in the CLI identity file. The deploy script now reuses the
  funded identity and reads the secret back via `stellar keys show`.
- `stellar contract restore/read/extend --key --durability` flag spelling
  **verified against the installed CLI** — correct as written.
- `deploy-and-shrink-ttl.sh` now runs end-to-end on testnet: deploys,
  initializes, scans, prints Healthy.
- Remaining soft spot: `trigger-eviction-wait.sh` writes scratch state to
  `/tmp/` files (works, slightly unclean).

### CI workflows (`.github/workflows/`) — self-contained, partly proven live

- `demo-scan.yml` and `demo-restore.yml` no longer depend on a contract
  `ttl()` (which cannot exist); both use the new off-chain reader
  `scripts/read-entry-ttl.py` (stdlib python: builds the
  `LedgerKey::ContractData` XDR, fetches it, reads the `liveUntilLedgerSeq`
  the RPC populates — direct TTL-key queries are rejected by soroban-rpc).
  Cross-validated against the live entry and the sentinel: **identical
  TTLs** (120,873 at first cross-check, later 120,794 — same
  `live_until_ledger` 4,704,624).
- `test-contract.yml` (new, `4a85eb8`): cargo test + wasm build on push and
  every PR. **Live run on the push: passed.**
- `demo-scan.yml`'s scheduled cron is proven to fire, but has **never once
  succeeded**: as of 2026-09-14 all 22 scheduled runs failed with
  `CONTRACT_ID repository variable is not set` (every run back to 2026-09-08).
  The single green run in the repo's history is a manual `workflow_dispatch`
  (`34832138820`, 2026-09-14T10:14Z), which shows the variable exists but is
  **not** evidence about the schedule — a correction to issue #5, whose closing
  comment reads it the other way. See the closeout pass above.
- `capture-restore-transcript.yml` (`585ebb6`) — closes the gap that made issue
  #2 uncloseable; see the closeout pass above for what it captures and why
  `demo-restore.yml` is left alone.

### Fixture manifest (`contracts.yml`) — proposal, unvalidated

- Declares the fixture (network, contract-id env var, WASM path, entry
  key + padded SCVal XDR, sentinel thresholds) for `action-state-watch`'s
  self-check. Since that repo is not public, the schema is a documented
  proposal rather than a validated contract (issue #4).

### Docs — real captures now

- `docs/surviving-soroban-state-archival.md` and
  `docs/setting-extend-ttl-boundaries.md` use **ledger-verified numbers**
  (parameters, rent denominators, fee plateaus) with the exact
  `getLedgerEntries` recipe (CONFIG_SETTING / ConfigSettingID 10) to
  reproduce them.
- The scan JSON example is now the **real capture** of the live entry
  (Healthy, 120,847 ledgers remaining at capture, size 72 B) with an
  explicit note that the Critical/Archived variants land when the entry
  decays and that the demo's accelerated thresholds (`--healthy-days 1
  --critical-days 1`) differ from the sentinel's defaults (30d/7d).
- `.transcripts/` holds the actual deploy transcript, sentinel scan JSON,
  watcher check, and off-chain TTL read — cross-checked against each other.
- README ties the pipeline together, links the transcripts, and includes
  the differentiation paragraph (SoroScope = gas/CPU profiling,
  Soroban-Guard = static security analysis; this suite = post-deployment
  TTL/archival monitoring).
- **README banner (Gap 1) now in place** (`f4a6f26`) — an earlier
  compressed-raster banner attempt was reverted at the owner's request
  (`main` reset to `b6c5840` and force-pushed, then the 1.8 MB handoff
  source in `images/` removed in `be6d10e`), so the banner was regenerated
  from scratch as an SVG: a hand-authored 1280×640 `assets/banner.svg`
  with the repo name as real `<text>` elements (`ARCHIVAL` in teal,
  `FIXTURES DEMO` in blue — pulled from the repo name itself so it can't
  drift), tagline "Simulating Soroban state archival on testnet" grounded
  in the README's own description, and a blue→teal shield on the dark
  `#111318` background. Rasterized to `assets/banner.png` (~53 KB,
  1280×640, under the 200 KB target) and referenced above the README
  title, centered per the confirmed approved-repo convention (`assets/`
  dir + `<p align="center">`, matched `karagozemin/Sub-Rosa`; see Review
  method table). Remote verified: GitHub's rendered README rewrites the
  `<img>` to the raw URL, which serves 200 `image/png` end-to-end.

### Hygiene — clean

- `.gitignore`: `.deploy/` (secret key + contract id) and `target/`
  correctly ignored; `git check-ignore` confirms; `.transcripts/` committed
  deliberately (real run output, no secrets — verified).
- `SECURITY.md` names the single key in the suite (the TESTNET-ONLY
  throwaway demo key) and states no other tool holds a key.
- `CONTRIBUTING.md` codifies the git workflow; `scripts/create-issue-backlog.sh`
  documents the issue-backlog generation.

## What is now proven by execution

1. **Live deployment on testnet** — contract
   `CAEDHSOD3TXIAZF2BZMMNX7A2OKBCVE4WU7A6RWTHGGHWHJXHEQUMAT4`, entry
   initialized, real explorer transaction. Healthy at ~120,790+ ledgers and
   decaying ~1 ledger per ~5s (observed value dropping across checks).
2. **Sentinel scan against real state** — Healthy band with the repo's
   thresholds; Critical-under-defaults behavior also observed live (the
   default 30d/7d thresholds flag a fresh ~7-day entry Critical, justifying
   the repo's explicit 1d/1d override).
3. **Off-chain TTL read == sentinel** — same `live_until_ledger_seq`, same
   TTL, to the ledger.
4. **`cargo test` green (5/5)** and the WASM build green — locally and in
   GitHub Actions (`test-contract.yml` passed on the push).
5. **The scheduled cron fires** — demo-scan has run on schedule every ~6 h
   without interruption; all 22 scheduled runs failed for the documented,
   expected reason (missing `CONTRACT_ID` variable), not a workflow bug. Note
   the corollary: *firing* is proven, *succeeding on the schedule* is not —
   the only green run is a manual dispatch.
6. **Scripts run end-to-end from a stranger's setup path** — the README's
   deploy → check flow was exercised as written.

## What is still pending (real time, not blockers)

1. **Critical → Archived → restore phases** — the entry is decaying in real
   time; Critical lands in ~6 days, Archived in ~7, then
   `run-full-pipeline.sh --wait-for archived` exercises the restore path.
   The transcripts for those phases will be appended when they land (issue
   #2). Until then, the docs state this plainly — no fabricated output.
2. **`demo-restore.yml` live run** — cannot be exercised until an entry is
   archived (or the owner runs it against the live entry, which exercises
   the extend + verify path only).
3. **`contracts.yml` validation** — issue #4 is *closed* as blocked, but the
   blocker has since cleared (`action-state-watch` is now public), so this is
   actionable again rather than done.
4. **Sentinel release binaries** — none published; consumers build from source.
   Issue #7 was closed as out-of-scope for this repo, but the promised redirect
   issue in `soroban-state-sentinel` was never opened, so the need is currently
   tracked nowhere.
5. **Remaining Gap-1 items** — none: the README banner, badges, GitHub topics
   (8 set) and the contrib.rocks question are all closed. The contrib.rocks
   block was added and then removed (`a2e0e9c` added badges; `91de157`/`94418aa`
   removed contrib.rocks) in favour of an honest "Maintainers"/"Contributors"
   section stating there are no outside contributors.

## Blocked items that need the repo owner (token permissions)

The environment token is refused (HTTP 403) for: Actions variables/secrets,
workflow dispatch, and the branch-protection API. To finish Gap 4 / Gap 5:

1. ~~Set the repository **variable** `CONTRACT_ID`~~ — **done.** The manual
   `demo-scan` dispatch on 2026-09-14T10:14Z read the live entry successfully
   (`band=Healthy ttl=33172`), which requires the variable. It has not yet been
   observed on a scheduled run — see the closeout pass.
2. Set the repository **secret** `TESTNET_THROWAWAY_SECRET_KEY` = the `S...`
   key in `.deploy/testnet-throwaway.secret` (TESTNET-ONLY throwaway).
3. Trigger `demo-scan.yml` (should report Healthy) and `demo-restore.yml`
   (extend+verify path now; restore path once archived); record the runs.
4. ~~Protect `main` with required checks~~ — **done** (issue #6): branch
   protection is active with the single required check `contract-tests`,
   `enforce_admins: true`, `required_approving_review_count: 0`, and force-push
   and deletion disabled. The earlier advisory here (never require `scan` or
   `restore` — both are schedule/manual-only and would deadlock every PR) was
   followed. Note the consequence: `enforce_admins: true` applies to the
   `GITHUB_TOKEN` too, which is why the transcript workflows must open PRs
   rather than push to `main`.

## Recommendations (what's left)

1. **Merge the transcript PRs when they land.** The capture side is now
   automatic for all three transcripts (Critical, Archived, restore); the only
   remaining manual step is the merge. Enabling auto-merge, or merging the
   scheduled PR when convenient, makes #2 genuinely self-closing. Do not close
   #2 by hand — let it close on the real data.
2. **Set the `TESTNET_THROWAWAY_SECRET_KEY` secret** (issue #5). It is the one
   thing still blocking `demo-restore.yml` and therefore the restore
   transcript; `CONTRACT_ID` is already set.
3. **Re-check the `demo-scan` schedule on the next cron.** Every scheduled run
   to date has failed and the only success was manual, so the schedule is
   unproven rather than "green" as #5's closing comment implies. A comment
   correcting that record would be worth posting.
4. **Reopen and action `contracts.yml` validation** (issue #4): its blocker
   (`action-state-watch` being public) has cleared, so the real schema can be
   read from `.github/workflows/self-check.yml` instead of guessed.
5. **Open the missing `soroban-state-sentinel` issue** for #7's release
   binaries — the redirect was declared but never made.
6. ~~Standalone-network rehearsal mode~~ — **done** (issue #3, PR #10): shipped
   as a standalone-network recipe (`docs/standalone-rehearsal.md`) rather than
   a `--rehearsal` flag. It drives a local throwaway node with a small
   `minPersistentTTL` so the arc completes in minutes. Rehearsal output is
   explicitly ephemeral and the doc states plainly that `.transcripts/` must
   only ever hold real testnet captures, so no `.rehearsals/` directory is
   needed.

## Summary

The completion pass converted most of the repo from "verified by
construction" to "verified by execution": the contract compiles and tests
green (after a real SDK-27 API fix), the scripts deploy and scan real
testnet state correctly (after three real execution fixes), CI has a green
live run, and the docs now carry real captured output. The demo is
currently mid-flight: deployed, Healthy, and decaying in real time toward
the Archived/restore demonstration. What remains is either real-time wait
(the decay), owner-side admin actions the environment token cannot perform
(variables/secrets, workflow dispatch, branch protection), or dependencies
outside this repo (`action-state-watch` publication). No known design flaws
remain unaddressed. The Gap-1 README banner item is now closed: after an
earlier raster-banner attempt was reverted at the owner's request, the
banner was regenerated as a hand-authored SVG with the repo name as real
`<text>` (so it can't drift out of sync) and rasterized to a ~53 KB PNG —
in the README, centered per the confirmed approved-repo convention, and
verified serving 200 over HTTPS. The remaining Gap-1 items are closed as well:
badges are in the README, topics are set (8), and contrib.rocks was replaced by
an honest "Maintainers"/"Contributors" section noting there are no outside
contributors. The closeout pass above records the one real fix it made (issue
#2's restore transcript), the three closed issues whose records are now wrong
or stale (#4, #5, #7), and the corrected entry state.
# Web Reactions Log (staging)

Public, append-only transparency log for Web Reactions counters. This repository
holds signed checkpoints and Bitcoin timestamps for the public reaction log. Used
with the public API and the open-source verifier, it lets anyone recompute the
counters and confirm the signed history was not silently rewritten.

**This is the staging log** for the test environment at
`https://api-staging.webreactions.app`, signed with its own key. Unlike the
production log ([web-reactions-log](https://github.com/khasky/web-reactions-log)),
it is **reset to genesis weekly** together with the staging database, so its
history is intentionally short-lived.

## What's here

Everything under `checkpoints/`, `ots/`, and `entries/` is written by the anchoring
bot. What each file is for:

**`checkpoints/` — signed tree heads (STHs)**

- `latest.json` — the single newest checkpoint (`tree_size`, `root_hash`, `ts`,
  `signature`), pretty-printed. **Overwritten every checkpoint.** The O(1) entry
  point consumers and the verifier read first; also the git-published view that is
  compared against the live API to catch a split view.
- `<YYYY-MM-DD>.ndjson` — the permanent **append-only archive**: one compact JSON
  line per checkpoint, one shard per UTC day — the browsable history of every STH
  ever signed. Today's shard is appended to and freezes once the day rolls over.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

The newest checkpoint is in both `latest.json` and the current shard on purpose (a
moving pointer plus an append-only archive), not by accident.

**`ots/` — Bitcoin timestamps (OpenTimestamps)**

A checkpoint's root is submitted to the OTS calendars, then matured into a proof
once it lands in a Bitcoin block, so an anchored checkpoint `<tree_size>` produces:

- `<tree_size>.pending.json` — interim calendar receipt right after submission
  (queued, not yet in a Bitcoin block); superseded once the proof matures.
- `<tree_size>.ots` — the matured OpenTimestamps proof (standard binary `.ots`)
  anchoring that checkpoint's root in a Bitcoin block.
- `<tree_size>.json` — self-contained sidecar for that proof: the signed STH + the
  Bitcoin block height, so a verifier needs nothing else to tie the `.ots` to a
  checkpoint. (The block height lives here, not in the binary `.ots`.)
- `latest.json` — pointer to the newest matured proof; overwritten as proofs mature.
- `.gitkeep` — empty marker so the directory survives a fresh/reset repo.

Not every checkpoint gets its own OTS proof — only the newest not-yet-submitted one
each time submit runs; the rest ride a consistency proof to an anchored one.

**`entries/` — reserved (empty)**

Holds only `.gitkeep`. The raw log leaves (individual reaction events) are **not**
published here — they are served by the public API at `/log/entries`, which the
verifier refetches to recompute the Merkle root against the signed checkpoint here.

Revocations are part of that same raw API log: account erasure and other public
corrections are append-only `op=4` leaves, exposed at `/log/revocations`. The
verifier checks that endpoint against the actual `op=4` leaves covered by the signed
root anchored here.

## Reading the commit history

Every commit here is made by the anchoring bot. The message says what it did:

| Commit message       | File written                      | What it means                                                                                                              |
| -------------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `add checkpoint 766` | `checkpoints/<YYYY-MM-DD>.ndjson` | a new Ed25519-signed tree head (STH) for `tree_size` 766 was appended — the substantive "a checkpoint was published" event |
| `update latest 766`  | `checkpoints/latest.json`         | the pointer to the newest checkpoint moved to 766 (the file the verifier reads)                                            |
| `ots submit 759`     | `ots/759.pending.json`            | checkpoint 759's root was submitted to the OpenTimestamps calendars; awaiting a Bitcoin block                              |
| `ots anchor 759`     | `ots/759.ots`                     | the proof matured — 759's root is now anchored in Bitcoin (the block height is recorded in the `ots/759.json` sidecar)     |
| `ots sidecar 759`    | `ots/759.json`                    | the self-contained sidecar for that proof (signed STH + block height)                                                      |
| `ots latest 759`     | `ots/latest.json`                 | the pointer to the newest matured proof moved to 759                                                                       |

`tree_size` is the cumulative number of log leaves — it only ever grows.

**Commit messages are informational only.** The verifier never reads them: it
recomputes everything from the file _contents_ here plus the public API's
`/log/*` endpoints. Read them to follow what the bot did; nothing depends on
their wording.

**Why the numbers look out of order.** Checkpoint `tree_size` values jump by
however many events landed in that hour (e.g. `742 → 754 → 759 → 766`), not by
one. And an OTS submit always anchors the _newest_ checkpoint not yet submitted,
so several submits walk newest → older (`766`, then `759`, …). Both are expected;
most intermediate checkpoints never get their own OTS proof and are tied to an
anchored one by consistency proofs instead.

**Editing this repository.** Docs (`README`, `LICENSE`, anything outside the data
directories) are safe to edit — the verifier ignores them and the bot never
touches them. The data directories — `checkpoints/`, `ots/`, and any published
`entries/` — are machine-generated: hand-editing them, force-pushing, or
rewriting history is exactly the tampering the verifier is built to catch (and
third-party mirrors preserve the real history). Don't edit them by hand.

## Verify

Use the open-source verifier (separate public repo):

```
git clone https://github.com/khasky/web-reactions-verifier
cd web-reactions-verifier
pnpm install
node src/verify.mjs \
  --api https://api-staging.webreactions.app \
  --repo https://raw.githubusercontent.com/khasky/web-reactions-log-staging/main \
  --pubkey <published staging Ed25519 key> \
  --ots
```

## Administrators

This repository holds only the published log data and its documentation — no
scripts or tooling. Operator maintenance (the weekly reset to genesis, scaffolding
a fresh empty layout) is performed by the backend automation and lands here as
normal bot commits, visible in the commit history like everything else.

Force-pushing or rewriting history in this repo is itself the tamper signal —
third-party mirrors (e.g. Software Heritage) preserve the real history.

## License

The log data in this repository is dedicated to the public domain under
[CC0 1.0 Universal](LICENSE) — copy, mirror, and verify it freely.

## Reset schedule

This staging log — and the entire staging database behind
`https://api-staging.webreactions.app` — is **fully reset to genesis every Monday
(around 03:00 UTC)**, and on demand. Everything in this repository is therefore
**ephemeral**: any checkpoints, proofs, or entries you see are wiped at the next
reset and the log is rebuilt from scratch. For anything durable, use the
production log at
[web-reactions-log](https://github.com/khasky/web-reactions-log).

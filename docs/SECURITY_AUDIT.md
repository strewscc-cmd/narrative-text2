# Quackville Security & Multiplayer Audit

Audit date: 2026-11-12  
Repository: strewscc-cmd/narrative-text2  
Scope: checked-in source files `index.html`, `teacher.html`, `database.rules.json`, `config.js`, and `README.md`. This is a source-code review, not a penetration test and not a verification of the live Firebase Console configuration.

## Executive summary

**Status: NOT READY for coin-staked Random Duel.**

The current game is a client-authoritative prototype. Player identity is a random browser-local identifier, coin balances are stored in localStorage, clients publish their own coin values to the shared leaderboard, and theft transfers are completed by client-side listeners. The checked-in Realtime Database rules permit broad writes to active room data. Adding a duel escrow system on top of this design would allow players to spoof identity, balances, match inputs, or outcomes.

Do not enable real coin stakes until the identity, balance authority, database rules, and trusted backend work are completed.

## Findings

### Critical — Active room data is broadly writable by unauthenticated clients

File: `database.rules.json`

The room-level write expression is:

```json
".write": "auth != null && auth.uid == 'XZU3OIVgrcTxn7HJsLZdVigqbap2' || !data.exists() || data.child('status').val() != 'ended'"
```

For an existing room whose status is not `ended`, the final condition evaluates true even for unauthenticated clients. Since Firebase permission grants cascade from parent paths, child-level restrictions cannot override a granted parent write. A student/browser can therefore write arbitrary descendants of an active room, including player data, board values, progress, theft events, and any future duel state.

The rule also allows an unauthenticated client to create a room when the room does not yet exist. The teacher UID branch allows that authenticated teacher to write the room even after it is ended.

**Impact:** no safe trust boundary for coin stakes, results, progress, or duel records.

**Required fix:** remove the broad parent `.write` grant; design narrowly scoped read/write rules for each existing game path and future duel path. Restrict trusted fields to a server identity/service account. Test rules in the Firebase Emulator Suite before deploying.

### Critical — Student identity is not authenticated

File: `index.html`

The multiplayer UID is generated with `Math.random()` and persisted in `localStorage.qv_uid`. It is not a Firebase Authentication UID and is not proof of a student's identity. A user can clear, replace, or copy this value.

**Impact:** clients can impersonate another player ID or create multiple identities; ownership rules based on this UID would not be secure.

**Required fix:** enable Firebase Anonymous Authentication (or another suitable student sign-in method) and use `firebase.auth().currentUser.uid` as the canonical player key. Migrate existing room membership carefully; do not silently mix old local IDs with authenticated UIDs.

### Critical — Coin balance is client-authoritative

File: `index.html`

The game initializes `save.coins` locally, merges data from `localStorage.qv`, persists changes back to localStorage, and sends `co: save.coins` from the browser in multiplayer player/board records.

**Impact:** a player can edit localStorage or submit an arbitrary `co` value. The teacher dashboard ranking reads the client-published board values, so rankings can be forged. An escrow implementation that trusts these values would be exploitable.

**Required fix:** make a trusted backend/ledger the sole authority for balances. Clients request rewards or transactions; the backend validates and updates balances. Publish leaderboard values from trusted balance records, not from browser-supplied coin fields.

### Critical — Existing theft transfer is client-to-client and forgeable

Files: `index.html`, `database.rules.json`

The challenger writes a record under `gN/thefts`. The target's browser listener calculates a transfer, edits its own local balance, and updates the theft record; the challenger then observes that update and edits its own local balance. The record contains client-supplied fields such as `from`, `to`, `pct`, `failed`, and `amt`.

**Impact:** any client with active-room write permission can forge theft records, fake amounts, mark events done, or interfere with another player's balance flow. This is not an atomic transaction and can desynchronize balances.

**Required fix:** replace client-to-client transfer authority with a trusted backend transaction. Preserve the current theft UI/gameplay if desired, but treat it only as a request; backend verifies eligibility, stakes, outcome, and applies one atomic ledger transaction.

### High — Teacher dashboard reads a leaderboard that is not authoritative

File: `teacher.html`

The dashboard builds ranking and total coin statistics from `rooms/{code}/board`. That board is populated by student clients through `mpSend()`.

**Impact:** dashboard totals and rankings are useful for classroom display but cannot be treated as verified financial records or audit evidence.

**Required fix:** source ranking and coin totals from backend-maintained balance/ledger records. Clearly distinguish legacy unverified balances from trusted balances during migration.

### High — Student progress is written by the client

File: `index.html`

`storySync()` writes story progress directly from the student browser to `rooms/{code}/progress/{uid}`. With the current broad room write rule, another client can also alter another student's progress.

**Impact:** progress, attempts, completion, and review flags are not protected from tampering.

**Required fix:** authenticate students and scope writes to their own progress. For high-integrity assessment, validate answers and completion server-side. Keep teacher override paths separately authorized.

### High — No trusted backend or Firebase deployment manifest found in repository root

Checked paths: `firebase.json`, `functions/package.json`, `functions/index.js`, and `package.json` were not found (404).

**Impact:** there is no checked-in Cloud Functions implementation or Firebase deployment manifest in the inspected paths. A trusted duel escrow/settlement system cannot be implemented securely as client-only JavaScript.

**Required fix:** add a Firebase Functions codebase, deployment configuration, emulator tests, and server-side handlers for invitations, escrow, match validation, disconnect expiry, settlement, and ledger writes. Verify the Firebase plan supports the chosen Functions deployment.

### Medium — Google Sheets endpoint/key are embedded in client code

Files: `index.html`, `teacher.html`

The Apps Script endpoint and shared key string are embedded in public client code. The student client posts progress directly with `mode: 'no-cors'`, so it cannot inspect the response to confirm that Sheets accepted the record.

**Impact:** the shared key is not a secret; requests may be replayed or forged if the Apps Script accepts it as sole authentication. Client UI may report a submission without verifying it.

**Required fix:** treat Sheets as a reporting/export destination, not a source of truth. Send trusted server-generated records from a backend or validate requests with stronger server-side authorization. Record and surface delivery failures.

### Medium — End Session is a client-orchestrated multi-step operation

File: `teacher.html`

The teacher browser writes the archive snapshot and then separately updates room status to `ended`. If the browser or network fails between these operations, the archive and room status can disagree. The current room rules also allow the broad student write described above while the room is active.

**Impact:** partial archive state or a race with student writes during session ending.

**Required fix:** move end-session/archive to a trusted backend transaction or carefully designed server workflow; atomically mark room ended before accepting further student mutations, then produce a final immutable snapshot.

## Compatibility observations

- Current multiplayer grouping uses `rooms/{code}/gN/players/{localUid}`.
- Current board is `rooms/{code}/board/{localUid}`.
- Current story progress is `rooms/{code}/progress/{localUid}`.
- Current theft events are `rooms/{code}/gN/thefts`.
- Teacher authentication checks a fixed Firebase Auth UID in the dashboard.
- The student game itself does not load Firebase Auth SDK or sign students in.
- The student app checks room status and applies client-side read-only behavior after `status === 'ended'`; this is useful UX but not a security boundary.

## Recommended remediation order

1. **Freeze financial features:** do not launch Random Duel stakes or treat current client-side coins as secure.
2. Add Firebase Anonymous Auth for students and migrate player keys from local random IDs.
3. Create a trusted coin ledger and migrate existing balances with an explicit teacher-reviewed baseline.
4. Add Cloud Functions for coin rewards, existing theft, duel invitations, escrow, game result validation, disconnect handling, and settlement.
5. Replace broad room write rules with least-privilege rules. Use Firebase Emulator Suite tests to prove unauthorized writes fail.
6. Move End Session/archive finalization to a trusted backend workflow.
7. Update teacher dashboard to read trusted balance and transaction records.
8. Test existing movement, grouping, story progress, announcements, theft, ranking, archive, and ended-session behavior before deploying.
9. Only then implement Random Duel Phase 1 and the first two mini-games.

## What this audit did not verify

- The currently deployed Firebase Console Realtime Database Rules; only the checked-in JSON file was inspected.
- Firebase Authentication provider settings, billing/plan, Cloud Functions availability, or deployed functions.
- The external Google Apps Script source and its actual authorization/validation behavior.
- Runtime behavior under multiple devices or load.
- A full penetration test or automated rules-emulator test.

**Bottom line:** the visual game and classroom multiplayer prototype can continue to be used for non-sensitive play, but current coin balances, theft outcomes, and board rankings should be considered untrusted until the authority model is migrated.

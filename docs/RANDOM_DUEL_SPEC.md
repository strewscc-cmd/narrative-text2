# Quackville Random Duel — Technical Specification

Status: Design approved; implementation not yet enabled  
Target: Existing Quackville room-based multiplayer game  
Database: Firebase Realtime Database  
Backend recommendation: Firebase Cloud Functions (trusted transaction authority)

## 1. Locked product decisions

- Match mode: real-time simultaneous; both students play at the same time.
- Challenger may invite any player in the same active room.
- The challenged player must explicitly accept.
- Duel type is selected randomly by the system after acceptance and stake validation.
- Available stakes: 15%, 25%, or 50%.
- Both players stake the same number of coins.
- The stake percentage is applied to the lower of the two players' eligible balances at acceptance time.
- Both stakes are reserved before the match begins. The winner receives the entire pot.
- A draw or technical cancellation refunds both stakes.
- A disconnected player gets a 10-second reconnect window; if they do not return, they forfeit.
- If both players disconnect, or the room ends, the duel is cancelled and stakes are refunded.
- Settlement must be idempotent: a match can affect balances at most once.

Example: A has 300 coins, B has 200 coins, stake is 25%. Each stakes 50 coins; the pot is 100 coins. The winner receives 100 coins.

## 2. Mini-game pool

All five games have equal selection probability (20% each):

| Game ID | Game | Initial target duration | Result |
|---|---|---:|---|
| duck_suit | Duck Suit (rock-paper-scissors) | 3 rounds | Most rounds won |
| tap_battle | Tap Battle | 7 seconds | Most valid taps |
| duck_race | Duck Race | 20–40 seconds | First valid finish |
| memory_match | Memory Match | 30–60 seconds | Most pairs, then fastest time |
| obstacle_dash | Obstacle Dash | 20–40 seconds | First valid finish |

The backend chooses and persists one game type per match. Clients must never independently randomize the game. If the selected game cannot start, cancel and refund rather than silently rerolling.

## 3. Match lifecycle

Invitation lifecycle:
- pending
- accepted
- rejected
- expired
- cancelled

Match lifecycle:
- reserving
- countdown
- playing
- reconnect_grace
- settling
- finished
- cancelled

Expected flow:
1. Challenger selects target and stake percentage.
2. Backend validates both players are present in the same active room and neither is already in a duel.
3. Backend creates an invitation with a short expiry (recommended 20 seconds).
4. Target accepts or rejects.
5. On acceptance, backend re-reads balances and room state, calculates equal stake, and reserves both stakes atomically.
6. Backend randomly selects one mini-game and creates a unique match ID.
7. Both clients receive the same match state and server-authoritative start/end timestamps.
8. Clients submit only their own game inputs.
9. Backend validates inputs/results, resolves win/draw/forfeit, and settles the pot once.
10. Final result and transaction ledger entry are persisted for the teacher dashboard.

## 4. Suggested Realtime Database shape

This is a proposed additive structure; do not deploy it until rules and backend handlers are implemented and tested.

```text
rooms/{roomCode}/
  duels/
    invitations/{invitationId}/
      from, to, stakePercent, status, createdAt, expiresAt
    matches/{matchId}/
      meta/
        status, gameType, startAt, endAt, stakeEach, pot,
        playerA, playerB, createdAt
      players/{uid}/
        ready, connected, lastSeen
      inputs/{uid}/{inputId}/
        type, payload, clientAt
      result/
        winner, reason, settled, settledAt
    history/{matchId}/
      playerA, playerB, gameType, stakeEach, pot,
      winner, reason, createdAt, settledAt
```

Do not store client-authoritative balances, winner declarations, settlement flags, or stake calculations as trusted values. Those fields are written only by the trusted backend.

## 5. Realtime synchronization

- Shared countdown and match timing use backend/server timestamps, not each device's local timer.
- Clients render at device frame rate and interpolate opponent movement between authoritative updates.
- For movement games, send compact state/input updates at a controlled rate (initial target: up to 10 updates/second per player).
- Tap Battle should batch input counts rather than write every tap as a separate database operation.
- Each input is scoped to the authenticated player's UID and active match.
- Backend validates input cadence, allowed action types, match window, and game-specific constraints.
- Client-side animation is presentation only; it cannot determine coins or final results.

## 6. Disconnect and room-end policy

- A trusted backend presence monitor detects disconnects and starts a 10-second grace window.
- Reconnection before expiry restores the last valid match state.
- If exactly one player fails to reconnect, that player forfeits.
- If both players fail to reconnect, cancel and refund both stakes.
- A room status of `ended` blocks new invitations and causes active matches to cancel/refund.
- A late reconnect cannot reverse an already settled result.
- Do not rely solely on browser `beforeunload`; use Firebase connection state and server-side timestamps/expiry checks.

## 7. Coin escrow and settlement

Use one trusted transaction authority for reservation and settlement.

Reservation:
- Re-read room status, player membership, active-duel locks, and both balances.
- Calculate `stakeEach = floor(min(balanceA, balanceB) * stakePercent / 100)`.
- Reject if stakeEach is zero or either player cannot cover it.
- Atomically reserve both stakes and mark both players as busy.
- Do not start countdown until reservation succeeds.

Settlement:
- Verify match is unsettled and the result is valid.
- Apply exactly one outcome: winner receives the whole pot; draw/cancel refunds both stakes.
- Write an immutable transaction record with match ID, players, game type, stake, pot, winner, reason, and timestamp.
- Mark match settled in the same trusted transaction boundary or use a recoverable idempotency key based on match ID.
- Release both players' active-duel locks.
- Rebuild/update leaderboard values from the canonical balance ledger.

## 8. Security requirements and current repository warning

The currently checked-in `database.rules.json` has a room-level `.write` rule that permits broad writes while a room is active (with a teacher UID override). That means a client may be able to write sensitive descendants while the room is active.

Before enabling Random Duel:
1. Remove broad student write permission at the room parent.
2. Define narrowly scoped student write paths for existing gameplay features.
3. Require authenticated, stable player identities; verify whether the current game uses anonymous Firebase Auth or a locally generated UID before choosing rules.
4. Permit students to write only their own allowed input/presence paths.
5. Restrict balances, escrow, winner, settlement, archives, and duel history writes to trusted backend/authorized teacher.
6. Keep teacher authorization server-verified; do not rely on hidden UI controls.
7. Test existing movement, joining, group membership, board, progress, theft, and end-session workflows against the new rules before deployment.

Important: changing the JSON file in GitHub does not itself deploy Firebase Realtime Database Rules. Deploy separately through Firebase Console or an authenticated Firebase CLI workflow after testing.

## 9. Teacher Dashboard requirements

Add a Duel Monitor section showing:
- active matches and both players' connection status;
- invitation and match status;
- game type, stake per player, and total pot;
- winner and outcome reason;
- transaction history and before/after balances;
- CSV export.

Teacher controls should allow cancelling a stuck match with a recorded reason. The dashboard must not provide arbitrary winner or balance editing. Ending a room must prevent new duels and safely cancel/refund active duels.

## 10. Implementation phases

Phase 0 — Compatibility and security audit
- Inspect player UID/authentication lifecycle.
- Trace canonical coin balance and leaderboard writes.
- Trace existing theft/pickpocket flow.
- Map all current RTDB writes and listeners.
- Design/test least-privilege rules without breaking existing gameplay.

Phase 1 — Duel foundation
- Trusted invitation/accept/reject/expiry handlers.
- Active-duel lock and equal-stake escrow.
- Server random selection and shared countdown.
- Disconnect grace period and room-end cancellation.
- Idempotent settlement and transaction ledger.

Phase 2 — First two games
- Duck Suit: simultaneous hidden choices, best of three.
- Tap Battle: simultaneous 7-second input window, server-validated tap batches.

Phase 3 — Advanced games
- Duck Race, Memory Match, Obstacle Dash.

Phase 4 — Teacher Dashboard and classroom testing
- Duel Monitor, history, audit ledger, CSV export.
- Test two separate devices/accounts, concurrent matches, disconnects, duplicate settlement calls, and End Session.

## 11. Definition of done

- Both players see the same match/game/timer state.
- No match begins without acceptance and successful escrow.
- Both stakes are equal and cannot exceed either balance.
- Clients cannot write trusted coin/result fields.
- One match can settle only once.
- Disconnect policy works at 10 seconds.
- Room end prevents and safely terminates duels.
- Teacher dashboard shows auditable match and coin history.
- Existing quests, movement, multiplayer, theft, ranking, and session archive continue to work.

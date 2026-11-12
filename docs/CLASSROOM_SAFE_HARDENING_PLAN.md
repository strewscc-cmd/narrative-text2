# Quackville — Safe Classroom Hardening Plan

Updated: 2026-11-12
Repository: strewscc-cmd/narrative-text2

## Goal

Keep Quackville fun and usable for classroom review. Coins are game currency for furniture and play only. Academic assessment remains in the teacher's Google Sheets workflow and must not depend on Quackville coin balances or leaderboard values.

This plan prioritizes preventing accidental data loss and cross-room interference without breaking the existing student experience.

## Important deployment mismatch

The live Firebase Console screenshot supplied on 2026-11-12 shows:

```json
{
  "rules": {
    "rooms": {
      "$code": {
        ".read": true,
        ".write": true
      }
    }
  }
}
```

The checked-in `database.rules.json` is different: it grants broad writes to active rooms through a conditional expression and has separate archive rules. Therefore, the repository file must not be assumed to represent the deployed rules. Any future deployment must compare the proposed rules with the live console first.

## Current compatibility constraints

- The student game currently creates a browser-local player ID; it is not a Firebase Authentication UID.
- Student multiplayer writes include player presence/position, group membership, board coin display, story progress, and theft events.
- The teacher dashboard writes announcements, teacher-managed progress overrides, group moves, session archives, and room status.
- The current teacher dashboard uses Firebase Authentication and checks a fixed teacher UID.
- Student coin balances are game state, not trusted records. Google Sheets remains the academic reporting destination.

Because students are not currently authenticated, changing room writes to require `auth != null` would break student multiplayer. Do not publish such a change until the student sign-in migration and tests are complete.

## Safe rollout

### Phase 0 — Preserve the working classroom game

1. Keep the currently deployed rules unchanged during preparation.
2. Export or back up the Realtime Database before any rules or data migration.
3. Keep coins, furniture, theft, and leaderboard strictly inside the game experience.
4. Do not use coins, leaderboard rank, or duel results as academic grades.
5. Do not launch coin-staked Random Duel until a trusted settlement design exists.

### Phase 1 — Establish identity without breaking old rooms

1. Enable Firebase Anonymous Authentication in the Firebase project.
2. Add student sign-in and wait for authentication before joining a room.
3. Use the Firebase Auth UID as the new canonical player ID.
4. Design a migration/transition for existing local IDs. Do not silently replace IDs while students are in active sessions.
5. Verify teacher sign-in still works and only the intended teacher UID can use teacher controls.

### Phase 2 — Separate room access and teacher actions

1. Define room membership and room ownership explicitly.
2. Allow students to read only the room(s) they joined.
3. Allow students to update only their own presence and permitted gameplay fields.
4. Keep announcements, group reassignment, teacher overrides, archives, and End Session teacher-only.
5. Avoid broad write permissions at a parent path: Firebase parent grants cascade to child paths.
6. Add database rules tests using the Firebase Emulator Suite before deployment.

### Phase 3 — Protect classroom continuity

Test on at least two student devices and one teacher device:

- Create a room and join with a valid code.
- Reject a student attempting to write to a different room.
- Reject a student attempting to overwrite another student's identity or progress.
- Confirm movement, online/offline presence, grouping, and leaderboard display.
- Confirm story attempts and teacher reset/extra-attempt controls still sync.
- Confirm announcements and group moves still work.
- Confirm End Session archives data and prevents further student changes.
- Confirm students can still buy and place furniture.
- Confirm Google Sheets academic records are still submitted as expected.

### Phase 4 — Optional competitive features

If Random Duel is added later, use a trusted backend for invitations, equal stakes, disconnect handling, result validation, and one-time settlement. Do not let either player's browser decide the winner's payout.

## Deployment checklist

- [ ] Database backup created and verified.
- [ ] Student Authentication enabled and tested.
- [ ] Teacher Authentication verified.
- [ ] Proposed rules reviewed against every student and teacher database operation.
- [ ] Emulator tests pass for allowed and denied operations.
- [ ] Existing classroom features pass the two-student/one-teacher test.
- [ ] Live Firebase Rules are compared with the reviewed file before publishing.
- [ ] Rollback copy of the previous live rules is saved.

## Current status

This document is a preparation plan, not a claim that production Firebase security has been changed. The live rules shown in the supplied screenshot remain permissive, and the current student client still depends on unauthenticated database access. No production rules have been published or changed by this plan.

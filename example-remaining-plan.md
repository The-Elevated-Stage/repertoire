<reference name="example-remaining-plan" version="1.0">

<metadata>
type: example
consumers: arranger, repetiteur
tier: 3
</metadata>

<sections>
- scenario
- complete-plan
- format-notes
</sections>

<section id="scenario">
<context>
# Example: Remaining Plan (Revision r1)

This example shows the complete format for a remaining plan produced after a consultation. The scenario: a background sync feature hit a blocker in Phase 3 when FCM (Firebase Cloud Messaging) proved incompatible with the project's offline-first requirement. The consultation determined WebSockets as the replacement approach, requiring Phase 3 revision and downstream Phase 4-5 adjustments.

**Original plan:** `background-sync-plan.md` (5 phases)
- Phases 1-2: Completed successfully (database layer, local sync engine)
- Phase 3: Blocked at Task 3.2 (FCM integration failed)
- Phases 4-5: Not started (notification UI, settings)

**Consultation outcome:**
- Phase 3 revised (FCM → WebSockets)
- Phase 4 adjusted (notification triggers changed)
- Phase 5 preserved unchanged
- One rollback task added (remove partial FCM scaffolding from Phase 3)
</context>
</section>

<section id="complete-plan">
<core>
## Complete Remaining Plan

```markdown
# Implementation Plan: Background Sync (Revision 1)

<!-- plan-index:start -->
<!-- verified:2026-02-20T14:32:00 -->
<!-- revision:1 -->
<!-- supersedes:background-sync-plan.md -->
<!-- phase:3 lines:38-112 title:"Real-Time Sync via WebSockets" -->
<!-- conductor-review:3 lines:113-138 -->
<!-- phase:4 lines:139-198 title:"Notification Integration" -->
<!-- conductor-review:4 lines:199-220 -->
<!-- phase:5 lines:221-267 title:"Settings & User Preferences" -->
<!-- conductor-review:5 lines:268-285 -->
<!-- plan-index:end -->

<!-- overview -->
## Overview

This remaining plan covers Phases 3-5 of the background sync feature. Phases 1-2
(database layer and local sync engine) completed successfully and are unaffected
by the consultation — the sync engine's internal API remains stable regardless of
the transport layer change.

Phase 3 has been revised to use WebSockets instead of FCM for real-time sync
signaling. This change was driven by FCM's incompatibility with the offline-first
architecture — FCM requires Google Play Services, which conflicts with the
requirement that sync work without cloud dependencies (dramaturg-journal entry
2026-02-15: "must work completely offline, cloud sync is a bonus not a requirement").
<!-- /overview -->

<!-- consultation-context -->
## Consultation Context

**Blocker:** Task 3.2 (FCM Integration) failed — FCM SDK initialization crashes
on devices without Google Play Services. The Conductor attempted three workarounds:
(1) graceful degradation check, (2) HMS fallback for Huawei, (3) polling fallback.
All failed to meet the offline-first constraint — each still required at least one
cloud dependency for the signaling path.

**Approach change:** Replaced FCM with a WebSocket-based signaling service.
WebSockets operate over standard TCP with no platform service dependency. The local
sync engine (Phase 2, completed) already has a transport-agnostic interface — only
the signaling adapter needs replacement, not the sync logic.

**Journal constraints that influenced this decision:**
- "must work completely offline" (dramaturg-journal, 2026-02-15) — binding constraint
- "cloud sync is a bonus not a requirement" (dramaturg-journal, 2026-02-15) — shapes priority
- "prefer platform-standard approaches" (arranger-journal, 2026-02-17) — surface choice, overridden by the offline-first constraint which takes precedence per the priority chain

**Completed work impact:**
- Phases 1-2: Verified unaffected — sync engine interface is transport-agnostic
- Task 3.1 (Sync Service Architecture): Verified unaffected — service lifecycle is transport-independent
- Task 3.2 (FCM Integration): Rolled back — partial FCM scaffolding removed
- Task 3.3 (Retry Logic): Preserved with revision — retry logic applies to WebSockets with adjusted timeout parameters

**Tasks removed:**
- Task 3.4 (FCM Token Management) `(REMOVED)` — no longer needed; WebSockets use session-based connections, not device tokens

**Tasks added:**
- Task 3.5 (WebSocket Connection Manager) `(NEW)` — connection lifecycle, reconnection, heartbeat
<!-- /consultation-context -->

<!-- phase-summary -->
## Phase Summary

| Phase | Title | Status | Dependencies |
|-------|-------|--------|-------------|
| 3 | Real-Time Sync via WebSockets | Revised | Phases 1-2 complete |
| 4 | Notification Integration | Adjusted | Phase 3 |
| 5 | Settings & User Preferences | Unchanged | Phase 4 |

Phases execute sequentially. Phase 3 begins with a rollback task to remove FCM
scaffolding before new WebSocket work starts.
<!-- /phase-summary -->

<!-- phase:3 -->
## Phase 3: Real-Time Sync via WebSockets

**Objective:** Replace the FCM signaling layer with a WebSocket-based approach that
operates without platform service dependencies.

**Prerequisites:** Phase 2's sync engine is complete and exposes a transport-agnostic
`SyncSignalAdapter` interface at `lib/services/sync/signal_adapter.dart`. The new
WebSocket implementation plugs into this interface.

### Rollback: Remove FCM Scaffolding

**Action:** Revert commits abc1234, def5678
**Scope:** Files modified: `lib/services/fcm/`, `android/app/build.gradle` (FCM dependency), `pubspec.yaml` (firebase_messaging)
**Reason:** Partial FCM integration left non-functional code and an unused dependency
**Post-revert state:** `lib/services/fcm/` directory removed, build.gradle and pubspec.yaml restored to Phase 2 completion state
**Verification:** `flutter build apk` succeeds, no FCM imports remain in codebase

### Task 3.1: Sync Service Architecture
*(No annotation — unchanged, paused Musician resumes)*

The sync service lifecycle management at `lib/services/sync/sync_service.dart`.
Handles service start/stop, background execution registration, and battery
optimization negotiation. This task was completed before the blocker and is
unaffected — service lifecycle is transport-independent.

### Task 3.2: WebSocket Signaling Adapter (REVISED)

**Original:** FCM integration adapter. **Revised:** WebSocket signaling adapter.

Implement `WebSocketSignalAdapter` conforming to the `SyncSignalAdapter` interface.
The adapter manages a WebSocket connection to the sync relay server at the
configurable endpoint stored in `lib/config/sync_config.dart`.

**Implementation details:**
- Connection URL: `wss://{server}/sync/{device-id}`
- Reconnection: exponential backoff starting at 1s, max 30s, with jitter
- Heartbeat: ping every 25s, timeout at 35s
- Message format: JSON with `type`, `payload`, `timestamp` fields
- Offline behavior: adapter reports disconnected state; sync engine falls back to
  local-only mode (already implemented in Phase 2)

**Files:** `lib/services/sync/websocket_signal_adapter.dart`
**Tests:** Unit tests for connection lifecycle, reconnection backoff, message parsing

### Task 3.3: Retry Logic for Failed Syncs (REVISED)

**Original:** Retry logic with FCM-specific error codes. **Revised:** Retry logic
with WebSocket-specific error handling.

Adjust the retry logic in `lib/services/sync/retry_handler.dart` to handle
WebSocket error codes (1006 abnormal closure, 1011 server error, 1013 try again
later) instead of FCM error codes. Timeout parameters change from 30s (FCM
delivery timeout) to 10s (WebSocket round-trip timeout).

**Files:** `lib/services/sync/retry_handler.dart`
**Tests:** Unit tests for each WebSocket error code path

### Task 3.5: WebSocket Connection Manager (NEW)

Implement connection manager handling multiple WebSocket channels (sync signaling,
presence updates). Manages connection pooling, authentication token refresh, and
graceful shutdown during app lifecycle events.

**Files:** `lib/services/sync/ws_connection_manager.dart`
**Tests:** Unit tests for connection pooling, token refresh flow, app lifecycle hooks
<!-- /phase:3 -->

<!-- conductor-review:3 -->
## Conductor Review: Post-Phase 3

- [ ] Rollback verification: no FCM imports remain (`grep -r "firebase_messaging"`)
- [ ] `flutter build apk` succeeds after rollback
- [ ] WebSocket adapter conforms to SyncSignalAdapter interface
- [ ] Reconnection backoff tested with simulated disconnections
- [ ] Offline fallback works — kill WebSocket server, verify local-only sync continues
- [ ] Integration: sync engine uses WebSocket adapter transparently (swap test)
- [ ] No regressions in Phase 1-2 test suites
<!-- /conductor-review:3 -->

<!-- phase:4 -->
## Phase 4: Notification Integration

**Objective:** Integrate local notifications with sync events to surface sync
status to the user.

**Prerequisites:** Phase 3's WebSocket adapter is operational. Notification triggers
now listen for WebSocket connection state changes rather than FCM message delivery
events.

### Task 4.1: Notification Service Setup

Set up `flutter_local_notifications` at `lib/services/notifications/notification_service.dart`.
Android notification channel: "Sync Status" with default importance.
The notification service subscribes to the WebSocket adapter's connection state
stream (exposed via `SyncSignalAdapter.connectionState`).

**State mapping:**
- Connected + syncing → silent (no notification needed)
- Disconnected > 5 minutes → "Sync paused — reconnecting"
- Reconnected after gap → "Sync resumed — X items updated"

**Files:** `lib/services/notifications/notification_service.dart`
**Tests:** Unit tests for state-to-notification mapping

### Task 4.2: Notification UI Components

Notification tap action opens the sync status screen.
In-app banner for active sync conflicts requiring user resolution.
Badge count on settings icon when sync is paused.

**Files:** `lib/widgets/sync/`, `lib/screens/sync_status_screen.dart`
**Tests:** Widget tests for banner display logic, badge count updates
<!-- /phase:4 -->

<!-- conductor-review:4 -->
## Conductor Review: Post-Phase 4

- [ ] Notifications display correctly for each connection state
- [ ] Tap action navigates to sync status screen
- [ ] No notifications during normal connected operation (silent path)
- [ ] Battery impact: notification service does not keep CPU awake
- [ ] Integration: end-to-end from WebSocket disconnect → notification → reconnect → dismiss
<!-- /conductor-review:4 -->

<!-- phase:5 -->
## Phase 5: Settings & User Preferences

**Objective:** Add user-facing settings for sync behavior and notification preferences.

**Prerequisites:** Phases 3-4 complete. Settings control the sync and notification
services implemented in prior phases.

### Task 5.1: Sync Settings Screen

Settings screen at `lib/screens/settings/sync_settings_screen.dart` with controls:
- Sync enabled/disabled toggle
- Sync frequency selector (real-time, 15min, 30min, 1hr, manual)
- Wi-Fi only toggle
- Clear sync data button (with confirmation dialog)

**Files:** `lib/screens/settings/sync_settings_screen.dart`, `lib/models/sync_preferences.dart`
**Tests:** Widget tests for all controls, preference persistence

### Task 5.2: Notification Preferences

Notification settings within the sync settings screen:
- Notifications enabled/disabled
- Quiet hours configuration (start/end time pickers)
- Notification sound selector

**Files:** `lib/screens/settings/notification_prefs_section.dart`
**Tests:** Widget tests for quiet hours logic, preference persistence
<!-- /phase:5 -->

<!-- conductor-review:5 -->
## Conductor Review: Post-Phase 5

- [ ] All settings persist across app restart
- [ ] Sync frequency selector actually changes sync behavior
- [ ] Wi-Fi only toggle tested with mobile data simulation
- [ ] Quiet hours respected — no notifications during configured window
- [ ] Full regression: Phases 1-5 integration test suite passes
<!-- /conductor-review:5 -->
```
</core>
</section>

<section id="format-notes">
<context>
## Format Notes

Key formatting conventions demonstrated in this example:

**Plan index:** Revision metadata (`revision:1`, `supersedes:`) appears only in remaining plans. The `verified` timestamp confirms the verification loop passed. Line ranges must be accurate after all edits — the index is generated last.

**Consultation context:** Unique to remaining plans. Covers the blocker, what was tried, what changed, and what journal constraints influenced the decision. The Conductor reads this instead of the full consultation journal.

**Phase numbering:** Original phase numbers preserved (3, 4, 5 — not renumbered to 1, 2, 3). Completed phases (1, 2) are excluded entirely.

**Task annotations:**
- No annotation on Task 3.1 (unchanged — Musician resumes)
- `(REVISED)` on Tasks 3.2, 3.3 (approach changed — new Musician)
- `(NEW)` on Task 3.5 (didn't exist before)
- `(REMOVED)` for Task 3.4 listed in Consultation Context, not in the phase section

**Rollback tasks:** Appear first in the first phase. Self-contained with commit hashes, file scope, expected post-revert state, and verification steps.

**Self-containment:** Each phase describes its prerequisites concretely — file paths, interface names, current state. A Copyist reading only Phase 4 can produce task instructions without referencing the superseded plan.

**Sentinel markers:** Every section wrapped in matching open/close HTML comments. Phase sentinels include the phase number (`<!-- phase:3 -->` / `<!-- /phase:3 -->`).
</context>
</section>

</reference>

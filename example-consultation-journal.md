<reference name="example-consultation-journal" version="1.0">

<metadata>
type: example
consumers: arranger, repetiteur
tier: 3
</metadata>

<sections>
- scenario
- complete-journal
- format-notes
</sections>

<section id="scenario">
<context>
# Example: Consultation Journal

This example shows the complete consultation journal produced during the same scenario as the remaining plan example — a background sync feature where FCM was replaced with WebSockets. The journal progresses through all four stages with checkpoint entries at each trigger point.

**Journal file:** `docs/plans/designs/decisions/background-sync/consultation-1-journal.md`
**Corresponding plan:** `background-sync-plan-r1.md`
</context>
</section>

<section id="complete-journal">
<core>
## Complete Journal

```markdown
# Consultation 1 Journal: Background Sync

## Stage 1 — Ingestion: Blocker Assessment

**Finding/Decision:** The blocker is FCM's dependency on Google Play Services.
Task 3.2 (FCM Integration) fails on devices without GPS — initialization crashes
with `MissingPluginException`. The Conductor attempted three workarounds: graceful
degradation check (still crashes before the check runs), HMS fallback (adds a second
cloud dependency), polling fallback (violates offline-first constraint).
**Rationale:** All three workarounds preserve the FCM signaling model. The root
issue is that FCM as an architecture requires cloud service availability, which
contradicts the user's core constraint.
**Impact:** This is not a Task 3.2 bug — it is an architectural incompatibility
between FCM and the offline-first requirement. Resolution requires replacing the
signaling approach, not fixing the FCM integration.
**Strength:** core

---

## Stage 1 — Ingestion: Journal Constraint Extraction

**Finding/Decision:** Key constraints identified from journal analysis (bootstrap
path):
1. "must work completely offline, cloud sync is a bonus not a requirement"
   (dramaturg-journal, 2026-02-15) — UNDERLYING CONSTRAINT: the app's value
   proposition is offline-first. Cloud features must degrade gracefully, not fail.
2. "prefer platform-standard approaches" (arranger-journal, 2026-02-17) — SURFACE
   CHOICE: expressed as a general preference, not tied to a specific technical
   requirement. Can be overridden when it conflicts with constraint #1.
3. "SQLite WAL mode for concurrent sync access" (arranger-journal, 2026-02-17) —
   UNDERLYING CONSTRAINT: the local database must support concurrent reads during
   sync. This is unaffected by the signaling layer change.
**Rationale:** Constraint #1 is the binding one. It was expressed as a core value
("the app's value proposition"), not a technical preference. Constraint #2 is
explicitly a preference ("prefer") and does not override #1.
**Strength:** mandatory

---

## Stage 2 — Impact Assessment: Impact Map

**Finding/Decision:** Impact classification:
- Phase 1 (Database Layer): UNAFFECTED — no dependency on signaling transport
- Phase 2 (Local Sync Engine): UNAFFECTED — SyncSignalAdapter interface is
  transport-agnostic by design
- Task 3.1 (Sync Service Architecture): UNAFFECTED — service lifecycle is
  transport-independent
- Task 3.2 (FCM Integration): INVALIDATED — the entire task is replaced
- Task 3.3 (Retry Logic): AFFECTED — retry error codes change with transport
- Task 3.4 (FCM Token Management): INVALIDATED — no longer needed
- Phase 4 (Notification Integration): AFFECTED — notification triggers change
  from FCM delivery events to WebSocket connection state
- Phase 5 (Settings): UNAFFECTED — settings control sync behavior, not transport
**Rationale:** The transport-agnostic interface in Phase 2 was a deliberate design
decision (arranger-journal, 2026-02-17: "abstract the signaling layer so transport
can change"). This containment worked — only the concrete adapter and its direct
consumers are affected.
**Impact:** Resolution scope is moderate: one task replaced, two tasks revised,
one task removed, one task added, one phase adjusted. Phases 1, 2, and 5 survive.
**Strength:** core

---

## Stage 2 — Impact Assessment: Scope Broader Than Expected

**Finding/Decision:** Phase 4 (Notification Integration) is more affected than
initially expected. The original Phase 4 tasks assumed FCM message delivery events
as notification triggers. With WebSockets, the trigger model changes from
"message received" to "connection state changed." This affects Task 4.1's event
subscription model.
**Rationale:** Traced the event flow: FCM `onMessage` callback → notification
trigger vs. WebSocket `connectionState` stream → notification trigger. Different
event model, different state mapping.
**Alternatives considered:** Keep FCM for notifications only (hybrid) — rejected
because it reintroduces the Google Play Services dependency.
**Strength:** core

---

## Stage 3 — Resolution: WebSocket Approach Selection

**Finding/Decision:** Chose WebSockets over three alternatives:
1. **WebSockets** (selected) — standard TCP, no platform dependency, bidirectional
2. **Server-Sent Events (SSE)** — simpler but server-to-client only, poor Android
   background support
3. **MQTT** — designed for IoT, adds a broker dependency
4. **Local polling** — simple but high battery impact at useful intervals
**Rationale:** WebSockets rank highest on the priority chain — compatibility
(works everywhere, no platform dependency), reliability (mature protocol, well-
supported in Flutter via `web_socket_channel`), efficiency (single persistent
connection vs. repeated HTTP requests).
**External input:** Gemini search confirmed `web_socket_channel` package is
actively maintained, supports all target platforms, and handles reconnection
patterns well.
**Strength:** core

---

## Stage 3 — Resolution: Journal Validation (Research Path)

**Finding/Decision:** CLEAR — WebSocket approach does not conflict with any journal
constraints.
- Constraint #1 (offline-first): WebSockets degrade gracefully — adapter reports
  disconnected state, sync engine falls back to local-only mode
- Constraint #2 (platform-standard): WebSockets are a web standard, not a
  platform-specific approach. The preference for "platform-standard" is satisfied.
- Constraint #3 (WAL mode): unaffected
**Rationale:** No conflicts found. The journal analysis subagent (research path)
confirmed no entry contradicts WebSocket adoption.
**Strength:** context

---

## Stage 3 — Resolution: Approach Impact Assessment

**Finding/Decision:** WebSocket adoption does not expand scope beyond the initial
impact map. The WebSocket adapter plugs into the same SyncSignalAdapter interface.
Connection manager (new Task 3.5) is self-contained.
**Rationale:** Traced the WebSocket adapter's integration surfaces — it implements
the same interface, produces the same event types (connected, disconnected,
message received), and stores no additional state beyond connection handle.
**Strength:** context

---

## Stage 3 — Resolution: Conductor Ground-Truth Validation

**Finding/Decision:** Conductor confirmed Task 3.1's service architecture is
transport-independent — "the SyncService starts and stops the adapter via the
interface, it doesn't know or care what transport the adapter uses."
**Rationale:** Validates that replacing the adapter does not require changes to
the service lifecycle.
**External input:** Conductor response via repetiteur_conversation table.
**Strength:** context

---

## Stage 3 — Resolution: Rollback Decision

**Finding/Decision:** Rollback Task 3.2's partial FCM work. Do not attempt
corrective modification.
**Rationale:** Task 3.2's FCM scaffolding is fundamentally incompatible — there
is no path from FCM adapter code to WebSocket adapter code that would be shorter
than starting fresh. Default to rollback per mandatory rules.
**Alternatives considered:** Refactor FCM adapter into WebSocket adapter (rejected
— different protocol model, no reusable code).
**Strength:** core

---

## Stage 4 — Verification: Pass 1 Results

**Finding/Decision:** Pass 1 (dual Opus + Gemini review) found one critical issue:
Phase 4 Task 4.1 did not specify the connection state → notification mapping.
The task said "subscribe to connection state" but did not define what states
produce what notifications.
**Rationale:** This would leave the Copyist guessing about notification behavior.
**Impact:** Added explicit state mapping table to Task 4.1.
**Strength:** core

---

## Stage 4 — Verification: Pass 2 Results

**Finding/Decision:** Pass 2 (narrowed scope — only Task 4.1 correction) confirmed
the state mapping is complete and internally consistent. No new issues found.
**Rationale:** Pass 2 scope was correctly narrowed to the single corrected task.
**Strength:** context

---

## Stage 4 — Handoff: Final Summary

**Finding/Decision:** Remaining plan r1 produced and verified. FCM replaced with
WebSockets. One rollback task, two revised tasks, one removed task, one new task.
Phase 4 adjusted for WebSocket event model. Phase 5 unchanged.
**Rationale:** All journal constraints satisfied. Impact contained by the
transport-agnostic interface. Verification loop passed (2 passes).
**Strength:** core
```
</core>
</section>

<section id="format-notes">
<context>
## Format Notes

Key conventions demonstrated in this example:

**Entry structure:** Every entry has Finding/Decision and Rationale (mandatory). Alternatives considered, Impact, External input, and Strength appear when applicable.

**Stage progression:** Entries are organized by stage (1-4) and topic. Multiple entries per stage are normal — Stage 3 (Resolution) typically has the most entries because each decision point gets its own entry.

**Constraint references:** Journal entries cite specific journal sources with dates. The format is `(journal-name, YYYY-MM-DD)` for traceability.

**Underlying vs. surface:** The constraint extraction entry explicitly labels each as UNDERLYING CONSTRAINT or SURFACE CHOICE. This distinction drives all downstream decisions.

**Checkpoint density:** This is a moderate-scope consultation. Larger consultations with cross-phase restructuring would have more entries, particularly in Stage 2 (impact assessment) and Stage 3 (resolution decisions per affected area).

**Append-only:** Entries are never edited or removed. If a decision is revised later in the consultation, a new entry records the revision with a reference to the original decision.

**Scope surprise:** The "broader than expected" entry in Stage 2 shows how to document scope discovery. The impact map was updated — the journal records both the finding and the map adjustment.

**Strength field:** Each entry includes a `Strength` annotation. The constraint extraction entry is `mandatory` (captures inviolable user constraints). Most analytical and resolution entries are `core` (standard decisions with rationale). Validation and confirmation entries are `context` (informational, no constraint weight). Downstream consumers use Strength to calibrate constraint extraction — `mandatory` violations are always errors, `core` violations require investigation, `context` items are informational only.
</context>
</section>

</reference>

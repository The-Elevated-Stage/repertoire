<reference name="journal-conventions" version="1.0">

<metadata>
type: shared-reference
consumers: arranger, repetiteur, dramaturg
tier: 3
</metadata>

<sections>
- purpose
- location-and-naming
- format
- entry-categories
- dramaturg-entry-types
- checkpoint-triggers
- journal-types
- lifecycle
</sections>

<section id="purpose">
<core>
# Journal Conventions

## Purpose

Decision journals are progressive external memory — they capture analysis, decisions, and reasoning as work progresses. They ensure no work is lost if context compaction occurs and provide an audit trail for future sessions.

Journals serve three audiences:
- **The current session** — if context becomes constrained, the journal preserves earlier findings and decisions for re-reading
- **Future sessions** — journal analysis subagents read all journals to prevent circular fixes and understand what was already decided or revised
- **The user** — the full reasoning trail for post-hoc review, especially important for auto-approved plans where the user did not review in real-time
</core>
</section>

<section id="location-and-naming">
<core>
## Location and Naming

All journals live in the feature's decisions directory:

```
docs/plans/designs/decisions/{feature-name}/
  dramaturg-journal.md              <- Dramaturg session
  arranger-journal.md               <- Arranger session
  consultation-1-journal.md         <- First consultation
  consultation-2-journal.md         <- Second consultation
  consultation-3-journal.md         <- Third consultation
```

Consultation journal numbering matches the remaining plan revision: `consultation-1-journal.md` corresponds to `{feature}-plan-r1.md`.
</core>
</section>

<section id="format">
<core>
## Format

Append-only markdown files. Entries are categorized and sequential. Entries are appended, never edited — the journal is a log, not a living document.

### Document Title Convention

Journal documents use the title format: `# Consultation N Journal: {Feature Name}`

For example: `# Consultation 1 Journal: Background Sync Implementation`

### Entry Template

```markdown
## [Stage/Phase]: [Topic]

**Finding/Decision:** [What was found or decided]
**Rationale:** [Why, including what constraints influenced this]
**Alternatives considered:** [What was rejected and why]
**Impact:** [What this affects in the plan]
**External input:** [What the Conductor/user said, if consulted on this point]
**Strength:** [mandatory | core | context]

---
```

The **Strength** field indicates constraint weight for downstream consumers:
- `mandatory` — user overrides, non-negotiable constraints. Highest constraint weight. Violations are always errors.
- `core` — standard decisions with rationale. Normal constraint weight. Binding but revisable with justification.
- `context` — informational, alternatives explored. Lowest constraint weight, informational only. Does not produce constraint violations.

Downstream consumers (particularly the Repetiteur's journal analysis) use the Strength field to calibrate constraint extraction. A `mandatory` entry produces an inviolable constraint; a `core` entry produces a binding-but-revisable constraint; a `context` entry is informational and does not constrain resolution.

Fields are flexible — not every entry needs every field. The mandatory fields are:
- **Finding/Decision** — what happened
- **Rationale** — why

Other fields are included when applicable. The **Strength** field should be included for any entry that records a decision or constraint — it may be omitted for purely informational entries where `context` is the obvious default. Brevity is fine for straightforward decisions; thoroughness is required for decisions that future sessions might question.
</core>
</section>

<section id="entry-categories">
<core>
## Entry Categories

Entries fall into three categories based on the type of information they capture:

### Goal/Use-Case Entries
Capture what the user wants to achieve. These are high-level intent statements.
- "User wants background sync to work even when app is closed"
- "Must support offline-first with eventual consistency"

### Decision Entries
Capture specific technical or approach decisions with rationale.
- "Chose FCM over SSE because SSE was reported unreliable on Android"
- "Set polling interval to 30 minutes due to Android platform minimum"

### Constraint Entries
Capture discovered limitations, platform behaviors, or non-negotiable requirements.
- "Android limits WiFi scans to 30-minute intervals"
- "SQLite WAL mode required for concurrent access"

For most journals, the category is implicit in the entry content — the journal analysis subagent infers category from context when processing entries. Dramaturg journals use an explicit `Category` field (see dramaturg-entry-types section). Goal/use-case entries are inviolable constraints regardless of the `Strength` field — they represent user intent at the vision level.
</core>
</section>

<section id="dramaturg-entry-types">
<context>
## Dramaturg Entry Types

Dramaturg journals use three distinct entry types with field names that differ from the standard entry template. Both conventions are valid — downstream consumers must handle both.

### Decision Entries

```markdown
## [Topic]

**Category:** [goal | use-case | decision | constraint]
**Decided:** [What was decided]
**User verbatim:** [The user's exact words, quoted]
**User context:** [What the user meant, interpreted by the Dramaturg]
**Alternatives discussed:** [Options explored with the user]
**Arranger note:** [VERIFIED | PARTIAL | UNRESEARCHED]
**Status:** [decided | revised | deferred]
**Strength:** [mandatory | core | context]
```

### Research Entries

```markdown
## [Topic]

**Question:** [What needed investigation]
**Tools used:** [Gemini, web search, etc.]
**Findings:** [What was found]
**Decision:** [What was decided based on findings]
**Arranger note:** [VERIFIED | PARTIAL | UNRESEARCHED]
**Status:** [decided | revised]
**Strength:** [mandatory | core | context]
```

### Tension Entries

```markdown
## [Topic]

**Requirements in tension:** [Which requirements conflict]
**Why they conflict:** [The nature of the conflict]
**Current resolution approach:** [How the tension is being managed]
**Status:** acknowledged
**Strength:** [mandatory | core | context]
```

Entries with `Arranger note: UNRESEARCHED` indicate decisions made without research backing — downstream consumers should treat these as lower-confidence decisions and consider independent verification during planning or resolution.

Entries with `Category: goal` or `Category: use-case` are inviolable constraints — treat equivalent to `Strength: mandatory` regardless of the explicit Strength value.
</context>
</section>

<section id="checkpoint-triggers">
<core>
## Checkpoint Triggers

Journals are written at defined trigger points, not ad-hoc. Each trigger represents a moment where significant state should be persisted.

### Arranger Checkpoint Triggers
- After each research phase decision
- After user approval/override at each gate
- After feasibility findings
- After each phase section is completed
- At finalization

### Repetiteur Checkpoint Triggers

**Stage 1 — Ingestion:**
- Initial assessment: understanding of the blocker
- Journal analysis findings: key user constraints identified, adjacent decisions flagged
- Gaps or clarifications requested from the Conductor

**Stage 2 — Impact Assessment:**
- Impact map: what is affected, what is isolated, rollback vs. correction classification
- Scope classification: how broad the resolution needs to be
- Any surprises — impact broader or narrower than the blocker suggested

**Stage 3 — Adaptive Resolution:**
- Each significant decision: approach chosen, alternatives rejected, journal constraints that influenced the choice
- Conductor dialogue: questions asked and responses received
- Rollback vs. corrective task decisions with reasoning for each affected piece of completed work
- Phase structure decisions for the remaining plan

**Stage 4 — Verification & Handoff:**
- Verification loop results: Pass 1 findings, fixes applied, Pass 2 results (if triggered)
- Final summary: what the remaining plan contains, what changed, what was preserved

### Context Threshold Checkpoints
Both Arranger and Repetiteur write state checkpoints at context thresholds (50%, 65%, 75%). These checkpoints contain enough state for the session to resume after compaction — current stage, progress, key findings, decisions made, remaining work.
</core>
</section>

<section id="journal-types">
<core>
## Journal Types and Their Treatment

Not all journals carry the same authority. The journal analysis subagent treats them differently based on whose voice they capture:

### User-Voice Journals (Dramaturg, Arranger)
These contain the user's actual words and explicit decisions. They are the primary source for underlying constraints.
- Decisions are **binding** — no session can override them without user escalation
- The "why" behind each decision is the constraint, not the specific choice
- A future Repetiteur or Arranger session must respect these constraints
- Dramaturg journals may contain `UNRESEARCHED` entries — items decided without research backing. These should be treated as lower-confidence decisions; downstream planners may verify them independently while still respecting the user's stated preference

### Autonomous Journals (Consultation)
These contain autonomous reasoning and decisions from the consultation session. They are context, not constraints.
- Decisions are **informative** — they tell future sessions what was already tried
- A future consultation can override a prior consultation's decision if new evidence warrants it
- Circular fix detection: if a proposed approach matches one a prior consultation rejected, flag it for scrutiny
- Oscillation detection: if a proposed approach reverses a prior consultation's decision, escalate to the user
</core>

<mandatory>User decisions from Dramaturg/Arranger journals are constraints that cannot be overridden without escalation. Autonomous decisions from consultation journals are context that can be overridden with justification. Never conflate the two.</mandatory>
</section>

<section id="lifecycle">
<core>
## Lifecycle

1. **Created** at the start of the session (Dramaturg research session, Arranger Phase 1, Repetiteur Stage 1)
2. **Appended to** throughout all phases/stages at defined checkpoint triggers
3. **Committed** alongside the plan during finalization/handoff
4. **Persists** after the session — NOT deleted or archived. Remains available for future sessions and user review.
5. **Cleaned up** by the Conductor after the full feature implementation is complete, along with the rest of the decisions directory
</core>
</section>

</reference>

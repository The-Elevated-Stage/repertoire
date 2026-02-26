<reference name="output-format" version="1.0">

<metadata>
type: shared-reference
consumers: arranger, repetiteur
tier: 3
</metadata>

<sections>
- document-structure
- plan-index
- revision-metadata
- consultation-context
- phase-sections
- conductor-review-sections
- sentinel-markers
- self-containment-rules
- task-annotation-markers
- file-location
- naming-convention
- rollback-tasks
</sections>

<section id="document-structure">
<core>
# Output Format

## Document Structure

Implementation plans follow a standardized structure that the Conductor and Copyist consume with identical logic. Both original plans (from the Arranger) and remaining plans (from the Repetiteur) use this format.

<context>
The template serves two consumers with different reading patterns. The Conductor reads overview, phase-summary, and conductor-review sections to understand the plan at a high level — what needs to happen, in what order, and what to verify. The Copyist reads phase bodies to generate task instructions — it needs self-contained implementation detail. Sentinel markers enable both consumers to selectively load only the sections relevant to their role.
</context>

```markdown
# Implementation Plan: [Feature Name] (Revision [N])  <!-- remaining-plan only: revision suffix -->

<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- revision:N -->                          <!-- remaining-plan only -->
<!-- supersedes:{feature}-plan-r{N-1}.md --> <!-- remaining-plan only -->
<!-- phase:N lines:NN-NN title:"[Phase Title]" -->
<!-- conductor-review:N lines:NN-NN -->
...
<!-- plan-index:end -->

<!-- overview -->
## Overview
[What remains to be built, current state of implementation.
 This plan must be self-sufficient: superseded plans are archived
 and no longer referenced by any downstream consumer. All relevant
 context from prior plans must be carried forward here.]
<!-- /overview -->

<!-- consultation-context -->        ← Remaining plans only
## Consultation Context
[What changed and why]
<!-- /consultation-context -->

<!-- phase-summary -->
## Phase Summary
[Phase map, dependencies, parallelization]
<!-- /phase-summary -->

<!-- phase:N -->
## Phase N: [Phase Title]
[Self-contained implementation detail]
<!-- /phase:N -->

<!-- conductor-review:N -->
## Conductor Review: Post-Phase N
[Review checklist]
<!-- /conductor-review:N -->

...repeat for all phases...
```
</core>
</section>

<section id="plan-index">
<core>
## Plan Index (Verification Index)

The plan-index block at the top of the file serves as both a table of contents and a lock indicator. Its presence confirms the plan passed finalization/verification.

```markdown
<!-- plan-index:start -->
<!-- verified:YYYY-MM-DDTHH:MM:SS -->
<!-- phase:3 lines:45-120 title:"Background Service Architecture" -->
<!-- conductor-review:3 lines:121-145 -->
<!-- phase:4 lines:146-230 title:"Notification Integration" -->
<!-- conductor-review:4 lines:231-260 -->
<!-- plan-index:end -->
```

Each entry maps a section to its line range. The Conductor uses these line ranges for selective reading — loading only the phase it needs without consuming context on the full document.

The `verified` timestamp is the Conductor's confirmation that verification occurred. A plan without a verified timestamp is unverified and must not be executed.
</core>

<mandatory>The verification index is generated after all verification passes complete. It must reflect the final, corrected state of the plan. Never generate the index before verification — the line numbers would be wrong after corrections.</mandatory>
</section>

<section id="revision-metadata">
<core>
## Revision Metadata (Remaining Plans Only)

Remaining plans include additional metadata in the plan-index:

```markdown
<!-- revision:2 -->
<!-- supersedes:location-features-plan-r1.md -->
```

- `revision:N` — which consultation produced this plan (1 = first consultation, 2 = second, etc.)
- `supersedes:{file}` — the plan this one replaces (moved to `superseded/`)

The Conductor reads these to know which revision it is operating from and what preceded it. This metadata is informational — the Conductor does not need to read the superseded plan.
</core>
</section>

<section id="consultation-context">
<core>
## Consultation Context Section (Remaining Plans Only)

A section unique to remaining plans, placed between the Overview and Phase Summary. Wrapped in its own sentinel markers for the Conductor to find via the index.

```markdown
<!-- consultation-context -->
## Consultation Context
...
<!-- /consultation-context -->
```

This section covers:
- The blocker that triggered consultation
- What the Conductor tried before invoking the consultation
- What approach changes the consultation made and why
- What completed work was affected (rollbacks, corrections, or verified unaffected)
- What journal constraints influenced the decisions
- References to the consultation journal for full detail

The Conductor uses this section to understand what changed without reading the full consultation journal or the superseded plan.
</core>
</section>

<section id="phase-sections">
<core>
## Phase Sections

Each phase is wrapped in sentinel markers with the phase number:

```markdown
<!-- phase:N -->
## Phase N: [Phase Title]
[Implementation detail]
<!-- /phase:N -->
```

Phase sections contain all implementation detail for the Copyist to produce task instructions. See self-containment-rules section for requirements.

**For remaining plans:** Original phase numbers are preserved. If the remaining plan covers Phases 3-6, it uses Phase 3, 4, 5, 6 — not Phase 1, 2, 3, 4. This preserves the Conductor's mental model and provides natural correspondence with the superseded plan.

**For remaining plans:** Completed, unaffected phases are not included — they exist in the superseded plan for reference only.
</core>
</section>

<section id="conductor-review-sections">
<core>
## Conductor Review Sections

Each phase is followed by its Conductor review section:

```markdown
<!-- conductor-review:N -->
## Conductor Review: Post-Phase N
[Review checklist]
<!-- /conductor-review:N -->
```

Review sections contain the Conductor's verification checklist for that phase — what to check, what integration surfaces to verify, what test suites to run. For remaining plans, review sections include any rollback verification items, corrective task validation, and cross-phase integration checks specific to the revised approach.
</core>
</section>

<section id="sentinel-markers">
<core>
## Sentinel Marker Convention

Sentinel markers use HTML comment syntax for machine readability without affecting rendered markdown:

| Marker | Purpose |
|--------|---------|
| `<!-- plan-index:start -->` / `<!-- plan-index:end -->` | Verification index bounds |
| `<!-- overview -->` / `<!-- /overview -->` | Overview section |
| `<!-- consultation-context -->` / `<!-- /consultation-context -->` | Consultation context (remaining plans only) |
| `<!-- phase-summary -->` / `<!-- /phase-summary -->` | Phase summary |
| `<!-- phase:N -->` / `<!-- /phase:N -->` | Phase section bounds |
| `<!-- conductor-review:N -->` / `<!-- /conductor-review:N -->` | Review section bounds |

Every sentinel marker must have a matching close marker. Header-sentinel consistency — the phase title in the sentinel must match the markdown header inside the section.
</core>
</section>

<section id="self-containment-rules">
<mandatory>
## Self-Containment Rules

Each phase section must pass the Copyist test: **Could a Copyist session reading only these lines produce complete, unambiguous task instructions?**

A self-contained phase section includes:
- Objective, prerequisites, implementation detail, integration points, expected outcomes, testing recommendations
- Frontend guidelines inlined when applicable (never in a separate block)
- All specific settings, file paths, component interactions, and error handling approaches decided — not left for the Copyist to figure out

**Additional self-containment for remaining plans:** Phase sections must include enough context about what preceded them (completed work, current state) that the Copyist does not need to reference the superseded plan. If Phase 4 depends on outputs from completed Phases 1-3, the Phase 4 section must describe those outputs concretely — file paths, exports, database state — not just "see prior phases."
</mandatory>
</section>

<section id="task-annotation-markers">
<core>
## Task Annotation Markers (Remaining Plans Only)

When the remaining plan includes tasks that correspond to tasks in the superseded plan, annotate them inline to signal the Conductor's required action during plan changeover:

| Annotation | Meaning | Conductor Action |
|------------|---------|-----------------|
| No annotation | Task unchanged, verified unaffected | Resume paused Musician |
| `(REVISED)` | Task modified — approach, scope, or dependencies changed | Close old Musician, generate new instructions via Copyist, launch fresh Musician |
| `(NEW)` | Task added — did not exist in superseded plan | Create new task row, generate instructions via Copyist, launch fresh Musician |
| `(REMOVED)` | Task eliminated — no longer needed | Close Musician, clean up task state |

**Example:**

```markdown
### Task 3.2: Implement Background Sync Service (REVISED)
...

### Task 3.3: Add Retry Logic for Failed Syncs
...

### Task 3.4: Platform-Specific Wake Lock Handling (NEW)
...
```

**Rules:**
- Annotations apply to tasks within phase sections, not to phases themselves
- Every task from the superseded plan's affected phases must have a clear disposition: present without annotation, present with `(REVISED)`, or listed as `(REMOVED)` in the Consultation Context section
- No task can silently disappear
- `(REMOVED)` tasks are not included in phase sections — they are listed in the Consultation Context with an explanation
</core>
</section>

<section id="file-location">
<core>
## File Location

The active plan lives in `docs/plans/designs/`. Only one plan is active at a time.

```
docs/plans/designs/
  {feature}-plan-r2.md              <- active (current)
  decisions/
    {feature-name}/
      dramaturg-journal.md
      arranger-journal.md
      consultation-1-journal.md
      consultation-2-journal.md
  superseded/
    {feature}-plan.md               <- original
    {feature}-plan-r1.md            <- first revision
```

The `superseded/` directory accumulates all prior plan versions. The Conductor never looks in `superseded/`. Superseded plans are read only during consultations (for context) and by the journal analysis subagent.
</core>
</section>

<section id="naming-convention">
<core>
## Naming Convention

| Plan | Filename |
|------|----------|
| Original | `{feature}-plan.md` |
| First consultation | `{feature}-plan-r1.md` |
| Second consultation | `{feature}-plan-r2.md` |
| Third consultation | `{feature}-plan-r3.md` |

The revision number is embedded in the plan metadata (verification index) so the Conductor always knows which revision it is operating from.
</core>
</section>

<section id="rollback-tasks">
<core>
## Rollback Tasks (Remaining Plans Only)

When the remaining plan includes rollback tasks, they appear as the first task(s) in the first phase of the remaining plan. The Conductor executes these before any new work begins — the rollback establishes a clean state for the revised approach.

Rollback tasks are explicit and self-contained:

```markdown
### Rollback: [Description of what is being undone]

**Action:** Revert commits [commit hashes]
**Scope:** Files modified: [file list]
**Reason:** [Why rollback is needed — connects to the blocker]
**Post-revert state:** [What the codebase looks like after revert]
**Verification:** [How to confirm the revert succeeded without regressions]
```

The Musician executing the rollback should not need to understand the consultation. The task must be fully self-contained.
</core>
</section>

</reference>

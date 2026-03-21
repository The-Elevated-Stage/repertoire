# Repertoire — Changes Needed for Arranger Alignment

**Date:** 2026-02-28
**Source:** `stagecraft/docs/working/2026-02-28-arranger-design-action-plan.md`
**Purpose:** Document all changes needed to repertoire shared contracts so that a separate session can execute them.

---

## Context

The Arranger design review (2026-02-28) identified format contract mismatches between the Arranger's specifications and the existing repertoire contracts. Key decisions made:

- **Plan output format:** Full Tier 2 hybrid (YAML frontmatter, `<sections>` index, `<section>` tags, authority tags within phases). The Arranger will produce this format; `output-format.md` must define it.
- **Journal format:** Flat markdown with a `Strength` annotation field (`mandatory`/`core`/`context`). The Arranger journal uses the same flat format as existing journals but adds semantic strength signaling.
- **Directory naming:** The canonical name for shared contracts is `repertoire/`. All references to `score-preparation/` or `shared-rules.md` in other skills are stale and will be updated there.

---

## Changes Required

### 1. `output-format.md` — Update to Tier 2 Hybrid Format

**Current state:** Defines sentinel-marker-only plans with no YAML frontmatter, no `<sections>` index, no `<section>` tags, no authority tags.

**Required changes:**

1. **Add YAML frontmatter schema.** The plan begins with YAML frontmatter:
   ```yaml
   ---
   title: "[Feature] Implementation Plan"
   date: YYYY-MM-DD
   type: implementation-plan
   tier: 2
   feature: "{feature-name}"
   design-doc: "docs/plans/designs/{design-doc-name}.md"
   ---
   ```

2. **Add `overview` and `phase-summary` entries to plan-index format.** Currently the plan-index example only shows `phase:N` and `conductor-review:N` entries. Add:
   ```
   <!-- plan-index:start -->
   <!-- verified:YYYY-MM-DDTHH:MM:SS -->
   <!-- overview lines:NN-NN -->
   <!-- phase-summary lines:NN-NN -->
   <!-- phase:1 lines:NN-NN title:"Phase Title" -->
   <!-- conductor-review:1 lines:NN-NN -->
   ...
   <!-- plan-index:end -->
   ```

3. **Add `<sections>` index specification.** After the plan-index, the plan includes a `<sections>` index listing all section IDs. This is a Tier 2 structural element providing fallback navigation alongside the primary sentinel/plan-index system.

4. **Add `<section>` tag conventions.** Each phase section is wrapped in `<section id="phase-N">` tags that coexist with sentinel markers. Sentinel markers remain the primary navigation mechanism; `<section>` tags provide structural validation and fallback.

5. **Add authority tag usage specification.** Within phase sections and conductor-review sections:
   - `<mandatory>` — non-negotiable constraints. Not modifiable by Conductor. Preserved verbatim by Copyist in task instructions.
   - `<guidance>` — recommendations. Conductor can adapt. Copyist can rephrase for task-level context. In conductor-review sections, these are suggestions, not blocking gates.
   - `<core>` — primary implementation content.

6. **Add danger file annotation format.** Phase sections may contain inline annotations marking known file conflicts:
   ```
   <!-- danger-file: path/to/file.dart shared-with="phase:3" -->
   ```
   The Conductor treats these as supplementary — self-discovery is primary.

7. **Add note distinguishing original plans from remaining plans.** Original plans (from Arranger) contain phase-level content without task headers. Remaining plans (from Repetiteur) contain task-level content with annotation markers (`(REVISED)`, `(NEW)`, `(REMOVED)`) because they reference the Conductor's existing task decomposition. This distinction is by design.

8. **Update consumer metadata.** Current: `consumers: arranger, repetiteur`. Add `conductor` and `copyist` — both consume this format.

9. **Update all examples.** The example plan structure should reflect the Tier 2 format with all elements present.

### 2. `journal-conventions.md` — Add Strength Annotation Field

**Current state:** Defines flat markdown entries with `Finding/Decision`, `Rationale`, `Alternatives considered`, `Impact`, `External input` fields.

**Required changes:**

1. **Add `Strength` field to entry template.** After `External input`:
   ```markdown
   **Strength:** [mandatory | core | context]
   ```
   - `mandatory` — user overrides, non-negotiable constraints. Carries strongest constraint weight for Repetiteur journal analysis.
   - `core` — standard decisions with rationale. Normal constraint weight.
   - `context` — informational deviations, alternative approaches tried. Weakest constraint weight, informational only.

2. **Document how Strength maps to constraint weight for Repetiteur parsing.** The Repetiteur's journal analysis extracts constraint strength. `mandatory` entries should be treated as inviolable. `core` entries are binding but potentially revisable. `context` entries are informational.

3. **Accommodate Dramaturg journal entry types.** The Dramaturg produces three entry types with its own templates:
   - **Decision entries** — Category (`goal`/`use-case`/`decision`), Decided, User verbatim, User context, Alternatives discussed, Arranger note (VERIFIED/PARTIAL/UNRESEARCHED), Status
   - **Research entries** — Question, Tools used, Findings, Decision, Arranger note, Status
   - **Tension entries** — Requirements in tension, Why they conflict, Current resolution approach, Status (`acknowledged`)

   `journal-conventions.md` should acknowledge that user-voice journals (Dramaturg, Arranger) may use skill-specific entry templates while maintaining the shared field set. The Dramaturg's field names (`Decided` vs `Finding/Decision`, `User verbatim` vs `Rationale`) are valid alternatives. Downstream consumers (Repetiteur journal analysis subagent) must handle both naming conventions.

4. **Add Dramaturg as explicit producer.** The Dramaturg produces journals following this convention (with skill-specific entry types).

5. **Ensure lifecycle rules remain clear.** Journals are NOT archived — they persist in `docs/plans/designs/decisions/{feature-name}/` until Conductor cleanup. This is already stated but should be reinforced given the Dramaturg's previous archival behavior.

### 3. `verification-rules.md` — Fix Path Reference

**Current state:** Line 161 references `score-preparation/output-format.md`.

**Required change:** Update to `repertoire/output-format.md`.

**Optional:** Consider whether the Arranger's extended structural verification items should be added to the shared contract:
- YAML frontmatter validation
- `<sections>` index completeness
- `<section>` tag presence matching sentinel markers
- Authority tag well-formedness

These are currently Arranger-specific extensions. Adding them to the shared contract would mean the Repetiteur's finalization also checks these elements, which is appropriate if the Repetiteur now produces Tier 2 format.

### 4. `priority-chain.md` — Verify Consumer Metadata

**No content changes needed.** Verify that consumer metadata lists all skills that reference the priority chain (Arranger, Repetiteur, Dramaturg, Conductor).

### 5. Consider Creating `hybrid-document-structure.md`

The Arranger design, Copyist, and other skills reference a "hybrid document structure" with Tier 1/2/3 definitions. This file doesn't exist in the repo. Multiple skills reference it:
- Tier 1: Human-focused narrative (Dramaturg design docs)
- Tier 2: Hybrid markdown with XML authority/structural tags (Arranger plans)
- Tier 3: All text inside authority tags (Copyist task instructions, decision journals)

If created in repertoire, it would be a shared contract defining:
- The three tiers and when each is used
- Authority tag allowlist (`<mandatory>`, `<guidance>`, `<core>`, `<context>`)
- Structural tag conventions (`<sections>`, `<section>`, `<reference>`)
- How tiers interact in the pipeline (Dramaturg produces Tier 1 → Arranger produces Tier 2 → Copyist produces Tier 3)

---

## Validation

After changes are made, verify:
- [ ] `output-format.md` example matches the Arranger's actual plan format specification
- [ ] `journal-conventions.md` entry template includes `Strength` field
- [ ] `verification-rules.md` path reference is corrected
- [ ] All consumer metadata is complete
- [ ] Examples (`example-consultation-journal.md`, `example-remaining-plan.md`) are updated to reflect format changes

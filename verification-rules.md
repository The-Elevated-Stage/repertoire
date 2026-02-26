<reference name="verification-rules" version="1.0">

<metadata>
type: shared-reference
consumers: arranger, repetiteur
tier: 3
</metadata>

<sections>
- mandatory-external-verification
- mental-implementation
- tool-selection
- research-conflict-resolution
- verification-summary
- structural-verification
</sections>

<section id="mandatory-external-verification">
<mandatory>
# Verification Rules

## Mandatory External Verification

These categories **must always** be verified through Gemini or web search, never assumed from training data. No exceptions, regardless of confidence level.

### Android and Cross-Device Services/Settings/Usage Patterns

Platform APIs change with OS versions. A setting that worked on Android 13 may be deprecated or restricted on Android 14. Training data is unreliable for version-specific behavior.

**Canonical example:** The WiFi scan interval limit — training data might say 5-minute intervals are possible, but Android actually limits scans to 30-minute intervals.

**Rule:** Every platform-specific service configuration, usage pattern, and limitation must be externally verified. No exceptions.

### Protocols and Patterns Not Already Implemented in the Project

If the project does not have working code for a pattern, do not trust training-data understanding. Version-specific gotchas, ecosystem compatibility issues, and platform constraints only surface through current research.

**Canonical example:** The FCM configuration that would have caused consistent app crashes.

**Rule:** If it is not already in the codebase, verify it externally. If the project already has working code for the pattern, no audit needed.

### Flutter Frontend Design Guidelines

Flutter's design ecosystem evolves between releases. Widget patterns, Material 3 guidelines, and recommended approaches change. External references produce stronger frontend guidance than training data alone.

**Rule:** No exceptions. Frontend design decisions are always verified against current external sources.

### Specific Configuration Values and Settings

When a plan specifies a concrete timeout, interval, batch size, or protocol setting, that value's validity must be checked against the target platform.

**Rule:** Every concrete setting value gets verified before entering the plan. "Use a 5-minute polling interval" needs verification that 5 minutes is actually achievable on the target platform.
</mandatory>
</section>

<section id="mental-implementation">
<core>
## "Mental Implementation" Verification

For features that touch components outside themselves, stress-test the approach by mentally walking through the implementation:

### Code Path Tracing via Read-Only Subagents

"If we add this service, trace the startup sequence — does it initialize before its dependencies?" Subagents explore existing code, follow the execution path, and report potential conflicts.

**Subagent prompt patterns:**
- "Trace the initialization sequence for [service]. List every file touched in order. Report potential conflicts with [new component]."
- "Simulate adding [feature] to the startup sequence. What initializes before it? What depends on it? What breaks?"
- These are templates adapted per context, not rigid scripts.
</core>

<mandatory>Read-only subagents only — no edits permitted. Subagent launch prompts must include explicit instruction: "Do not use Write, Edit, or Bash commands that modify files. This is a read-only planning session."</mandatory>

<core>
### Mocked Analysis via Gemini

Write a pseudo-implementation of a critical integration point and send it to Gemini for review. "Here is how I think [component] would initialize — what issues do you see?" Gemini catches ordering problems, missing error handling, and platform-specific gotchas that code-path tracing alone might miss.

### Cross-Component Dependency Checks

When Task A will modify a file that Task B also depends on, verify the integration contract. Will Task A's changes break Task B's assumptions? These findings feed directly into Conductor checkpoint sections as explicit verification items.

### When Mental Implementation Is NOT Needed

Simple implementations, wholly new files, changes isolated within a single component. The trigger is **external interaction** — if the change talks to or affects something outside its own scope, verify it.
</core>
</section>

<section id="tool-selection">
<core>
## Tool Selection

Best tool for each job, no rigid hierarchy:

| Tool | Best For |
|------|----------|
| **Gemini** (`gemini-search`, `gemini-query`) | Feasibility checks, setting validation, protocol verification, platform-specific questions, reviewing mocked implementations |
| **Web search** (`brave-search`) | Current documentation, known issues, community discussions, version-specific changelogs |
| **Explore subagents** | Codebase analysis, code path tracing, architectural understanding. Read-only — no edits. |
| **General-purpose subagents** | Multi-source synthesis, protecting main context from dead-end research |
| **WebFetch** | Reading specific URLs that research points to |

### Inline vs. Subagent Decision

- **Inline (main session):** Quick Gemini queries, single-source lookups, "just need to check this one thing." Fast, low overhead.
- **Subagent (delegated):** Multi-source investigation, code path tracing, speculative research that might hit dead ends. Protects main context. **All mental implementation verification runs in subagents.**

### Truth Hierarchy

When sources conflict: **Official documentation (web) > Gemini analysis > Training data.**
</core>

<context>
For extreme conflicts, conduct additional research focused on user-provided discussions online: forums, issue trackers, Stack Overflow threads, GitHub issues. These surface real-world implementation failures that official documentation and Gemini analysis may not capture.
</context>
</section>

<section id="research-conflict-resolution">
<context>
## Research Conflict Resolution

When sources contradict each other, prioritize real-world practitioner experience:

1. Official documentation establishes the baseline
2. Gemini analysis validates and extends understanding
3. Community discussions (forums, issue trackers, GitHub issues) surface practical failures not captured by official docs
4. Training data is the lowest priority — it may be outdated or wrong for version-specific behavior

If a proposed approach seems theoretically sound but community discussions report consistent failures, treat the community evidence as authoritative. Developers experiencing real problems in production outweigh theoretical correctness.
</context>
</section>

<section id="verification-summary">
<core>
## Verification Checklist Summary

Before any approach enters the plan, confirm:

- [ ] All Android/cross-device services/settings externally verified via Gemini or web search
- [ ] All unimplemented protocols/patterns checked via Gemini — no unverified protocol enters the plan
- [ ] All Flutter frontend guidelines validated against current external sources
- [ ] All specific configuration values (timeouts, intervals, batch sizes) verified against target platform
- [ ] Cross-component interactions stress-tested via mental implementation (read-only subagents)
- [ ] Mocked analysis for critical integration points sent to Gemini for review
- [ ] Any research conflicts resolved using truth hierarchy
</core>
</section>

<section id="structural-verification">
<core>
## Structural Verification

In addition to the content-focused checks above, every plan must pass structural integrity verification. These checks confirm the plan's mechanical correctness — that the document is well-formed and navigable by downstream consumers.

**Structural check items:**

- **Sentinel marker pair matching** — every opening sentinel marker (e.g., `<!-- overview -->`) has a corresponding closing marker (`<!-- /overview -->`). No orphaned markers.
- **Header-sentinel consistency** — section headers match their enclosing sentinel markers. A sentinel labeled `phase-3` wraps content that is actually Phase 3.
- **Verification index accuracy** — the `plan-index` at the top of the file correctly references every phase section with accurate line ranges. Line ranges must match the actual content positions after all edits.

The canonical definitions for what these structural elements look like (sentinel syntax, index schema, marker format) are in `score-preparation/output-format.md`. This section defines **what to check**; output-format.md defines **what correct looks like**.
</core>
</section>

</reference>

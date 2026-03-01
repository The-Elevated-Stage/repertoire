<reference name="priority-chain" version="1.0">

<metadata>
type: shared-reference
consumers: arranger, repetiteur, conductor, dramaturg
tier: 3
</metadata>

<sections>
- priority-ordering
- applying-the-chain
- examples
</sections>

<section id="priority-ordering">
<core>
# Priority Chain

## Architectural Trade-Off Ordering

When choosing between technically valid approaches that satisfy the same user constraints, use this priority ordering to resolve trade-offs:

```
compatibility > reliability > efficiency > security > performance
```

| Priority | Meaning |
|----------|---------|
| **1. Compatibility** | Works on all target platforms and integrates with existing components. An approach that fails on one platform is not a valid approach. |
| **2. Reliability** | Consistent, predictable behavior under real-world conditions. An approach that works 95% of the time loses to one that works 99% of the time, even if the 95% approach is faster or more elegant. |
| **3. Efficiency** | Resource usage, complexity, implementation effort. Simpler approaches win when reliability is equal. Fewer moving parts means fewer failure points. |
| **4. Security** | Data protection, input validation, access control. Prioritized after efficiency because most security requirements are non-negotiable constraints (captured in journals), not trade-off decisions. When security IS the trade-off, it ranks here. |
| **5. Performance** | Speed, throughput, latency. The lowest priority because performance issues are usually discoverable during implementation and can be optimized after the feature works. Architectural decisions that sacrifice compatibility or reliability for performance are almost always wrong. |
</core>
</section>

<section id="applying-the-chain">
<core>
## Applying the Chain

The priority chain resolves decisions **between approaches that all satisfy user constraints.** It does not override user decisions — if the user explicitly chose a less compatible approach for stated reasons, that decision is captured in journals and is binding.

The chain applies when:
- Multiple approaches clear journal validation (no constraint conflicts)
- The approaches differ on trade-off dimensions (one is more reliable but less performant)
- No journal entry expresses a user preference between the specific approaches

The chain does NOT apply when:
- A user decision in the journals directly addresses the trade-off
- Only one approach satisfies the constraints (no trade-off to resolve)
- The difference between approaches is negligible on all dimensions
</core>
</section>

<section id="examples">
<context>
## Examples

**FCM vs. WebSockets for push notifications:**
- Compatibility: both work on Android — tie
- Reliability: FCM is the platform-standard approach, more reliable for background delivery — FCM wins
- Result: FCM, unless the user's journal constraint makes reliability moot (e.g., "must work without Google services")

**SQLite vs. PostgreSQL for local storage:**
- Compatibility: SQLite is embedded, no external dependency — SQLite wins
- Result: SQLite for local storage, unless user constraints require features only PostgreSQL provides

**Polling vs. WebSockets for real-time updates:**
- Compatibility: both work — tie
- Reliability: polling is simpler with fewer failure modes — polling wins at this tier
- But: if the user constraint is "must update within 1 second," reliability of polling at that interval may be lower than WebSockets — reconsider
- Result: depends on the specific user constraint. The chain is a tiebreaker, not a rigid formula.

The chain provides a default ordering, not a mandate. When the specific context makes a lower-priority dimension more important (e.g., performance IS the feature), document the reasoning for deviating from the chain in the journal.
</context>
</section>

</reference>

@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Show Claude's assumptions about a phase before planning. Users can correct misconceptions early.

The key difference from discuss-phase: This is ANALYSIS of what Claude thinks, not INTAKE of what the user knows. The workflow does not write a file. It uses conversation to prompt discussion.
</purpose>

<process>

<step name="validate_phase" priority="first">
Phase number: $ARGUMENTS (need)

**If argument missing:**

```
Error: Phase number required.

Usage: /gsd:discuss-phase <phase-number> --assumptions
Example: /gsd:discuss-phase 3 --assumptions
```

Exit workflow.

**If argument provided:**
Make sure that the phase exists in the roadmap:

```bash
cat .planning/ROADMAP.md | grep -i "Phase ${PHASE}"
```

**If phase not found:**

```
Error: Phase ${PHASE} not found in roadmap.

Available phases:
[list phases from roadmap]
```

Exit workflow.

**If phase found:**
Parse the phase details from the roadmap:

- Phase number
- Phase name
- Phase description/goal
- Any scope details in the roadmap

Continue to analyze_phase.
</step>

<step name="analyze_phase">
Use the roadmap description and project context to identify assumptions across five areas:

**1. Technical Approach:**
What libraries, frameworks, patterns, or tools would Claude use?
- "I would use X library because..."
- "I would follow Y pattern because..."
- "I would structure this as Z because..."

**2. Implementation Order:**
What would Claude build first, second, third?
- "I would start with X because it is foundational"
- "Then Y because it depends on X"
- "Finally Z because..."

**3. Scope Boundaries:**
What does Claude's interpretation include and exclude?
- "This phase includes: A, B, C"
- "This phase does NOT include: D, E, F"
- "Boundary ambiguities: G could go either way"

**4. Risk Areas:**
Where does Claude expect complexity or challenges?
- "The tricky part is X because..."
- "Potential issues: Y, Z"
- "I would watch out for..."

**5. Dependencies:**
What does Claude assume exists or needs to exist?
- "This assumes X from previous phases"
- "External dependencies: Y, Z"
- "This will be used by..."

Be honest about uncertainty. Mark assumptions with confidence levels:
- "Fairly confident: ..." (clear from the roadmap)
- "Assuming: ..." (reasonable inference)
- "Unclear: ..." (could go multiple ways)
</step>

<step name="present_assumptions">
Show the assumptions in a clear format that is easy to scan:

```
## My Assumptions for Phase ${PHASE}: ${PHASE_NAME}

### Technical Approach
[List assumptions about how to implement]

### Implementation Order
[List assumptions about sequencing]

### Scope Boundaries
**In scope:** [what's included]
**Out of scope:** [what's excluded]
**Ambiguous:** [what could go either way]

### Risk Areas
[List anticipated challenges]

### Dependencies
**From prior phases:** [what's needed]
**External:** [third-party needs]
**Feeds into:** [what future phases need from this]

---

**What do you think?**

Are these assumptions accurate? Let me know:
- What I got right
- What I got wrong
- What I'm missing
```

Wait for user response.
</step>

<step name="gather_feedback">
**If user provides corrections:**

Acknowledge the corrections:

```
Key corrections:
- [correction 1]
- [correction 2]

This changes my understanding significantly. [Summarize new understanding]
```

**If user confirms assumptions:**

```
Assumptions validated.
```

Continue to offer_next.
</step>

<step name="offer_next">
Show the next steps:

```
What's next?
1. Discuss context (/gsd:discuss-phase ${PHASE}) - Let me ask you questions to build comprehensive context
2. Plan this phase (/gsd:plan-phase ${PHASE}) - Create detailed execution plans
3. Re-examine assumptions - I'll analyze again with your corrections
4. Done for now
```

Wait for user selection.

If "Discuss context": Note that CONTEXT.md will include any corrections discussed here.

If "Plan this phase": Continue with the understanding that the assumptions are understood.

If "Re-examine": Return to analyze_phase with the updated understanding.
</step>

</process>

<success_criteria>
- The phase number matches the roadmap.
- The assumptions cover five areas: technical approach, implementation order, scope, risks, dependencies.
- Assumptions have confidence levels where appropriate.
- The user sees the "What do you think?" prompt.
- Claude acknowledges user feedback.
- Claude offers clear next steps.
</success_criteria>

---
name: stabilizer
description: Use when a design or plan has been produced and needs scope assessment before implementation — especially when the plan introduces new abstractions, services, enums, modes, or multi-phase architectures. Auto-triggers in audit-only mode after brainstorming produces a design. Manual /stabilizer for commit-level output. After this skill completes, the stabilizer-gate skill runs automatically to verify output quality.
category: planning
---

# Stabilizer

Translate a design into two lanes — a minimal deterministic slice (Lane A) and the preserved vision (Lane B) — so the Architect decides what ships now vs. later. Advisory audit, never auto-rewrite.

**Core law:** Constraint precedes capability. Capability without constraint creates entropy. Constraint without capability creates brittleness. The order matters.

**Role model:** The Stabilizer Engineer — removes optionality, reduces branches, collapses enums, protects invariants, ships the 80% solution first. Not because vision is wrong, but because sequencing matters.

## Modes

| Mode | Trigger | Output | Rewrites plan? |
|------|---------|--------|----------------|
| **Audit** (default) | Auto after brainstorming/plan | Lanes + flags + decision prompt | Never |
| **Commit** | Manual `/stabilizer` | Implementation steps, PR scope, diffs | Only Lane A, only after user chooses |

**Audit mode is a lens, not a gatekeeper.** It labels and asks. It never blocks, recommends Lane A as morally superior, or quietly minimizes your vision.

## Step 1 — Classify the Problem

Before splitting lanes, classify **the problem as stated**, not the vision. The vision may cross boundaries; the problem may not. A single-file bug is FEATURE even if the brainstorming vision includes cross-boundary work. Classification determines how aggressively Lane A is compressed.

| Class | When | Lane A behavior | Lane B behavior |
|-------|------|-----------------|-----------------|
| **FOUNDATION** | Infra, safety, data integrity, auth, rate limiting, migrations, abuse prevention | Full rigor. SHOULD items get promoted. | Preserved. Interfaces designed now. |
| **INTEGRATION** | Cross-boundary (pipeline+DB+API+UI, auth flows, distributed) | Must include boundary contract tests + rollback | Ambitious but expressed as seams/interfaces only |
| **FEATURE** | Product iteration, UI, new endpoints, user-facing | Minimal slice prioritized. Most Lane B items → MAY | Preserved verbatim |
| **EXPERIMENT** | Spikes, research, prototypes, eval runs | Maximum learning speed. Prove one thing. | Just notes for later |

Inspired by [NASA SWE-020 software classification](https://swehb.nasa.gov/display/SWEHBVD/SWE-020+-+Software+Classification) — tailoring rigor to risk class, not applying uniform heavyweight process.

## Step 2 — Split Into Two Lanes

### Lane A — Minimal Deterministic Slice (MDS)

Answer: *"If I had 1 hour, what would I ship?"*

- **Objective:** Make the observed failure impossible (not reduced — impossible)
- **Boundary:** What single change, guard clause, constraint, reorder, or veto fixes this?
- **No new abstractions** unless required by an explicit MUST invariant
- **Invariants/contracts:** What must be true 6 months from now? Only those get rigor now
- **Rollback:** How do you undo this if it's wrong?
- **Tests:** Reproduction test + regression test, nothing more

### Lane B — System Vision (preserved)

<HARD-CONSTRAINT>
MUST: Lane B is the brainstorming output copied verbatim (formatting-only changes allowed). It is NEVER replaced with a minimized rewrite, summary, or "simplified version." The Architect's vision lives here untouched.
</HARD-CONSTRAINT>

Apply [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) tags to Lane B items without rewriting them:

- **MUST** — Absolute requirement. Goes into Lane A.
- **SHOULD** — Strong recommendation with valid exceptions. Phase 2 candidate.
- **MAY** — Truly optional. Phase 3+.

## Step 3 — Run Gate Checks

Scan the plan for these patterns. Output as RED / YELLOW / GREEN flags.

Each flag includes: (1) the exact sentence(s) in the plan that triggered it, (2) the smallest patch to resolve it.

| Gate | Question | RED if... | YELLOW if... |
|------|----------|-----------|--------------|
| **Invariant clarity** | What MUST be true 6 months from now? | No invariants identified | Invariants exist but aren't tested |
| **Smallest fix exists** | Is there a single change that makes failure impossible? | Yes, but plan doesn't include it | Smallest fix is in plan but buried under larger work |
| **New surface area** | Does plan add enums, modes, tiers, flags, services? | Adds surface area for FEATURE/EXPERIMENT class | Adds surface area for FOUNDATION/INTEGRATION class |
| **Capability before constraint** | Does plan add ability before enforcement? | New capability with no validation/guard | Capability and constraint in same phase but constraint is later |
| **Stability unproven** | Can you prove this works before expanding? | No test/metric for stability before Phase 2 | Test exists but doesn't cover load/edge cases |

Inspired by [Deming's PDSA cycle](https://deming.org/explore/pdsa/) — prove stability before expanding. And [Lean A3](https://www.lean.org/lexicon-terms/a3-report/) — if the problem doesn't fit on one page, the scope is wrong.

## Step 4 — Present Decision

End with a forced choice. Never skip this.

```
STABILIZER AUDIT
================
Classification: [CLASS] — [1-sentence why]

Lane A (smallest shippable slice):
  Objective: [make X impossible]
  Diff boundary: [files/functions]
  Invariants: [what must hold]
  Rollback: [how to undo]
  Tests: [repro + regression]

Lane B (preserved vision):
  [verbatim brainstorming output]
  MUST: [items] → already in Lane A
  SHOULD: [items] → Phase 2 candidates
  MAY: [items] → Phase 3+

Flags:
  RED: [list with exact plan quotes + smallest patch]
  YELLOW: [list with exact plan quotes + smallest patch]
  GREEN: [confirmations]

Decision required:
  → Ship Lane A only
  → Ship Lane A + specific SHOULD items: [list them]
```

**The skill never recommends one option as superior.** It presents both with their trade-offs. The Architect decides.

**No conclusion statements.** Do not write "Items X and Y are not on the table" or "Lane A is the clear choice given the flags." RED flags are information for the user's decision, not grounds for the skill to make the decision. Present flags → present options → stop.

## Anti-Override Constraints

These are non-negotiable MUST rules that prevent the Stabilizer from killing the Architect:

1. **Lane B is sacred.** Never summarize, minimize, or rewrite the vision.
2. **Never moralize.** "Lane A is the smallest shippable slice" — not "Lane A is the right choice."
3. **FOUNDATION class gets full rigor.** Don't compress infrastructure problems into feature-sized slices.
4. **Classification is explicit.** Always state the class and why. Never silently apply FEATURE-class compression to a FOUNDATION problem.
5. **Decision is always the user's.** The skill outputs a choice. It never auto-selects.
6. **Audit mode never rewrites.** It labels, flags, and asks. That's it.

## Handoff

After completing the audit, invoke the **stabilizer-gate** skill. It verifies output quality on the constraints this skill is statistically weakest at: classification accuracy, quote fidelity, patch specificity, abstraction creep, and neutrality. Do not skip this step.

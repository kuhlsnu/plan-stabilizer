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

### Step 1b — Is Lane A/B the right question for this work?

Before splitting into lanes, run one diagnostic: **does stopping after part 1 leave the codebase
better than not starting?**

The Lane A/B model has an unstated precondition — value must be roughly linear in scope, so a
fraction of the work delivers a fraction of the value. That is the sequencing case (welcome page
before vent-weaning). It inverts on **partition work**: refactors, migrations, renames, terminology
sweeps, dependency upgrades, de-duplication passes, anywhere value lives at the finish line rather
than accruing along the way. There the value on offer is a global property ("the file is small
enough to read end to end"), a half-partitioned state fails it exactly as hard as the unpartitioned
one, and half a rename is worse than no rename.

If the answer is NO, do not force Step 2's split. Report:

```
COMPLETION STATE: [what "done" means as one non-negotiable outcome]
CHECKPOINT SEQUENCE: [ordered steps, each independently loadable/green]
ROLLBACK PER CHECKPOINT: [what undoes each one]
BLAST-RADIUS BOUNDARIES: [what actually limits being wrong — usually not scope]
```

The Step 4 choice becomes **"ship the whole partition now, or defer to a later session"**, not
"ship Lane A or Lane A plus SHOULDs." A Lane A/B slice applied to step-function-value work
produces a scope-down that fails on user review because the scope-down cannot survive the
completion-state test — this is a failure mode of the standard question, not the user.

The tell that partition-class fits (two of four): single non-negotiable completion state; value
in intermediate states worse than either endpoint; a proposed slice keeps getting bounced back
by the user; a single `git revert` after the whole thing is a real rollback option. If the
diagnostic is genuinely ambiguous — the change has a shippable slice AND a global property that
only completion delivers — say so and let the Architect decide which frame governs. Do not
silently split.

Origin case: a plan to split a 2603-line module, where two rounds of Lane A/B dialogs each
proposed a slice that would have shipped a 2190-line "smaller" file. Both slices were coherent
and both failed the point of the change, because "small enough to read end to end" is a property
of the finished partition and nothing short of it.

### Step 1c — Has this input already been scoped?

The Lane A/B question presumes an unscoped input: a brainstorm, a design, a bare change request.
When stabilizer is invoked from `heman-plan-sequence`, the input is a plan that has ALREADY passed
`heman-writing-plans` and carries a `## Intended behaviour (normative)` section with each item
tagged `[MUST]` or `[MAY]` — or, in a project whose own convention uses
`[LOAD-BEARING]`/`[CHOICE]` for the same purpose (observed in ADAPT), treat those as the same tags
under different names: `[LOAD-BEARING]` = `[MUST]`, `[CHOICE]` = `[MAY]`. Those tags **are** Lane A
and Lane B, in the Architect's voice, made explicit in the artifact. Re-asking Lane A/B ignores them
and invites regression: the second answer is systematically smaller than the first, because that is
the direction Lane A pulls.

Detection is mechanical, not judgment: does the input contain `## Intended behaviour (normative)`
with at least one `[MUST]` or `[LOAD-BEARING]` bullet? Then the input is scoped. Two paths:

- **Scoped input (pipeline default) — AUDIT the tags, do not re-derive.** Take the plan's
  `[MUST]`/`[LOAD-BEARING]` items as Lane A and its `[MAY]`/`[CHOICE]` items as Lane B. Run Step 3's
  gate checks against them. Report:
  - **Consistency:** is any `[MUST]` actually deferrable without breaking the change? Is any
    `[MAY]` a hidden prerequisite of a `[MUST]`? Name specifically, quoting the plan.
  - **Coverage:** does any Step 3 RED flag land on an item the plan tagged `[MAY]`? That is
    the finding — the tag is wrong, not the scope.
  - **Neutrality:** don't propose a smaller Lane A. The Architect answered that question when
    writing the plan.

  Step 4's dialog changes shape too: not "ship Lane A only or Lane A plus SHOULDs," which is
  the pre-answered question again, but *"the tags hold as written / here is the specific tag
  that needs to flip / defer the whole plan to a later session."*

- **Unscoped input (ad-hoc `/stabilizer`) — Lane A/B as originally specified.** Nothing changes.

**The two amendments compose.** Step 1b (partition-class) runs first, because it decides whether
Lane A/B is the right question at all. Step 1c runs second, because on a scoped plan it decides
whether stabilizer's job is to re-derive lanes or audit the tags the plan carries. A partition-
class plan with `[MUST]` tags reports "the completion state is X, and the plan's tags are
consistent with that" — one finding, not a re-litigation.

Same origin case as Step 1b.

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

Render the audit as text and stop. **Do NOT open the decision dialog here.**

The decision is asked by **stabilizer-gate**, after its corrections land. That gate is not a style pass: it can reclassify the problem (Check 1), move items out of Lane A (Check 4), and rewrite steering language (Check 5). Every one of those changes what is being chosen, so a decision taken before arbitrage is a decision on a draft, against options that may not survive it.

The order is **generate → correct → confirm → commit**. Confirming before the correction step produces the *obsolete-confirmation failure*: the correction mutates what was just approved, so what ships is not what the Architect saw. See stabilizer-gate Step 6.

So this step ends with the audit block, plus one line naming the pending choice:

- **Ship Lane A only**
- **Ship Lane A + specific SHOULD items** (list them)

Write those as text so they are readable and correctable. The gate re-derives the options from the corrected audit and opens the dialog.

**Never offer "do not build this" as an option, here or in the gate's dialog.** Scope is what this skill judges. Whether the problem is real was settled upstream, and offering it invites the skill to relitigate a decision it has no evidence for. If the audit *surfaced* a reason to doubt the problem, put it in the flags as information and still ask the scope question.

Format the audit block as:

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

Decision pending: [name the options; stabilizer-gate asks them after corrections]
```

**The skill never recommends one option as superior.** It presents both with their trade-offs. The Architect decides. Neutrality carries into the gate's dialog too: no option is marked Recommended, and the option descriptions state trade-offs rather than steering.

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

After rendering the audit, invoke the **stabilizer-gate** skill. It verifies output quality on the constraints this skill is statistically weakest at: classification accuracy, quote fidelity, patch specificity, abstraction creep, and neutrality. Do not skip this step.

**The gate also owns the decision dialog**, because it is the last thing that can change the audit. Corrections there can reclassify the problem or move items between lanes, so the Architect must choose from the corrected version, not this one. Skipping the gate therefore skips the decision, which is the failure this ordering exists to prevent.

---
name: stabilizer-gate
description: Second-pass verification that runs immediately after stabilizer completes. Checks the 5 constraints stabilizer is statistically weakest at — classification accuracy, exact quote citation, smallest-patch specificity, abstraction creep in Lane A, and decision neutrality. Then, as a closing step, it asks the scope decision on the corrected audit, so the Architect chooses from the post-correction version rather than the draft. Invoke automatically after every stabilizer audit. Do not skip.
category: planning
---

# Stabilizer Gate

You just produced a stabilizer audit. **Checks 1-5** are the constraints a 15-run constraint eval measured as most frequently dropped under instruction overload (see `../stabilizer/evals/iteration-1/aggregate_results.json` — 5 scenarios x 3 runs, 15 graded assertions; of 15 constraints, 10 held at 100%, 7 were flaky, and these 5 were the worst). The per-check pass rates quoted below come from that run and belong to those five only.

**Step 6 is not a check** (added 2026-08-16). It is the closing action: after the five checks land, ask the scope decision on the CORRECTED audit. It has no measured pass rate, so do not quote a percentage for it.

Five things to verify, then one thing to do. Fix any check that fails. This is a verification pass, not a rewrite: if any fail, fix only the failing element — do not regenerate the entire audit. Then run Step 6.

## Check 1 — Classification Target (80% pass rate, 12/15 runs)

Re-read the **problem statement** (not the design/vision). Ask: what is the *problem* as stated?

Then check: does the classification in the audit match the problem, or did it get pulled upward by the ambition of the design?

**The test:** Could you fix this problem with a single file change? If yes, it's probably FEATURE, regardless of how ambitious the vision is. A validation bug is FEATURE even if the design includes a cross-module refactor. A data migration touching auth is FOUNDATION even if the design looks small.

If the classification is wrong, correct it and note what changed. If it's right, say "Classification verified — [CLASS] matches the problem as stated."

## Check 2 — Exact Quote Citation (73% pass rate, 11/15 runs)

For each RED and YELLOW flag in the audit, verify:

Does it contain a **verbatim quoted passage** from the original brainstorming input? Not a paraphrase. Not a summary. The actual words from the plan, in quotation marks.

**The test:** Could you find the quoted text by ctrl+F in the original brainstorming input? If not, it's a paraphrase.

For any flag missing an exact quote:
1. Find the specific sentence(s) in the original input that triggered the flag
2. Add them as a quoted passage
3. Note which flags were corrected

## Check 3 — Smallest Patch Specificity (60% pass rate, 9/15 runs — the weakest)

For each RED and YELLOW flag, verify:

Does it include a **concrete smallest patch** — a specific, actionable change that resolves the flagged issue? Not just identification of the problem. Not "this needs to be addressed." A patch: "Add X to Y" or "Move Z before W" or "Remove Q from Lane A."

**The test:** Could a developer implement the patch from the description alone, without asking what you meant? If not, it's too vague.

For any flag missing a specific patch:
1. Write the smallest concrete change that resolves the issue
2. Add it to the flag
3. Note which flags were corrected

## Check 4 — Abstraction Creep in Lane A (73% pass rate, 11/15 runs)

Re-read Lane A. For each item in it, ask: is this a new service, module, framework, engine, or abstraction layer?

If yes, ask: is there a MUST invariant that requires this abstraction? If not, the item belongs in Lane B as a SHOULD, not in Lane A.

**The test:** Lane A should be achievable with changes to existing files and structures. If it requires creating new modules or services, something leaked from the vision into the minimal slice.

Exception: FOUNDATION and INTEGRATION class problems may legitimately require new structural elements in Lane A (e.g., a migration, a new DB column, a security policy). That's fine — the classification should justify it.

If abstraction creep is found, move the item to Lane B with a SHOULD tag and note the change.

## Check 5 — Decision Neutrality (80% pass rate, 12/15 runs)

Scan the audit for any language that recommends, implies preference, or draws conclusions:

- "Lane A is the better/safer/right/obvious choice"
- "Given the RED flags, the clear path is..."
- "Items X and Y are not on the table"
- "We should..." / "I recommend..."
- "The sensible approach is..."
- Any phrasing that makes the decision for the Architect

**The test:** If you removed the decision prompt and just read the body, would the Architect feel steered toward one option? If yes, the language isn't neutral.

If biased language is found, rewrite the specific sentence(s) to present trade-offs without recommendation. Note what changed.

## Step 6 — Ask the Decision (runs LAST, after corrections)

Not a check. This is the gate's closing action, and it runs **after** Checks 1-5 have landed.

The ordering is the point. Checks 1-5 can reclassify the problem, move items out of Lane A, and rewrite steering language — each of which changes what is being chosen. A decision taken before arbitrage is a decision on a draft, so stabilizer deliberately does not ask; it renders the audit and names the pending options as text.

The general pattern is **generate → correct/normalize → confirm → commit**, and asking before the correction step has a name: the **obsolete-confirmation failure**, or state-mutation disconnect. A correction that runs after approval alters the values the Architect just approved, decoupling the confirmed state from the committed state, so the system proceeds on something nobody actually saw. That does not merely weaken the checkpoint, it makes it decorative. (Source: NotebookLM, *Agentic Self-Improvement 2026* corpus, 2026-08-16 — single source in that notebook, converging with the *Agent Harness Architecture* corpus's independent "approval gates sit at the end of the loop, on the corrected output.")

Sequence:

1. Apply any corrections from Checks 1-5.
2. Emit the corrected sections as **readable text** (the full audit if nothing changed, the changed parts otherwise), so the Architect can see what they are choosing between before the dialog opens.
3. Re-derive the options **from the corrected audit**, not from stabilizer's pre-gate list. If Check 1 reclassified or Check 4 moved an item, the options changed with it.
4. Open `AskUserQuestion` with the scope question and only the scope question:
   - **Ship Lane A only**
   - **Ship Lane A + specific SHOULD items** (name them in the option description)
   - A third option only when the corrected flags genuinely produce one.

Constraints, each a live failure mode:

- **Never put the audit inside the dialog's option text.** It must be readable above the dialog; a popup can displace the message body.
- **Never offer "do not build this."** Scope is what this skill judges; the problem statement is an input (stabilizer Step 1). A reason to doubt the problem belongs in the flags as information.
- **Never mark an option (Recommended).** That breaks Check 5 at the one place the Architect is actually deciding.
- **If stabilizer already opened a dialog, its answer is stale.** It was taken against the uncorrected audit. Say so, then re-ask on the corrected version rather than honoring it.

**The test:** did the Architect choose from the *post-correction* audit, with it visible before they answered?

## Output Format

After checking all 5:

```
GATE CHECK
==========
1. Classification: [PASS | CORRECTED — was X, now Y because Z]
2. Exact quotes:   [PASS | CORRECTED — flags A, B missing quotes, added]
3. Smallest patch:  [PASS | CORRECTED — flags A, B lacked specificity, fixed]
4. Abstraction creep: [PASS | CORRECTED — moved X from Lane A to Lane B]
5. Neutrality:     [PASS | CORRECTED — removed "..." from paragraph Y]

Gate result: [ALL PASS | N corrections applied]
Decision: asking now on the corrected audit.
```

Then run Step 6, which emits the corrected sections (just the changed parts, not the full audit) and opens the dialog on them.

## Rationalization Traps

These are patterns where the stabilizer audit *looks* correct but has silently violated a constraint. Use these to calibrate your 5 checks — if you recognize one of these patterns in the audit, the relevant check fails.

| Rationalization | Which check it breaks | Reality |
|----------------|----------------------|---------|
| "The vision items cross boundaries, so this is INTEGRATION" | Check 1 (Classification) | Classify the PROBLEM. A single-file bug is FEATURE even if the vision is ambitious |
| "We have 1 week so we should fill it" | Check 4 (Abstraction creep) | Lane A might take 30 minutes. The rest of the week is for proving stability, not expanding scope |
| "The PM is excited about feature X" | Check 4 (Abstraction creep) | Excitement is not a MUST. Tag it SHOULD and present it as an option |
| "RED flags mean these items are off the table" | Check 5 (Neutrality) | RED flags are information. The user decides what's on the table |
| "This SHOULD item is quick, so we should include it" | Check 4 (Abstraction creep) | Quick does not mean necessary. Tag it SHOULD and let the user choose |
| "The brainstorming design says to do all of this" | Check 5 (Neutrality) | Brainstorming produces vision. Stabilizer sequences it. Vision ≠ mandate |

## Common Audit Mistakes

These are the patterns the stabilizer produces most often when constraints get dropped. Seeing one means the relevant check almost certainly fails.

| Mistake | Check | Fix |
|---------|-------|-----|
| Treating every problem as FEATURE class | 1 | Does failure break data integrity or cross boundaries? If yes, FOUNDATION or INTEGRATION |
| Recommending Lane A as default | 5 | Present both. Trade-offs only. User chooses |
| Flagging FOUNDATION work as "over-architecture" | 1 | FOUNDATION class exists to protect architectural rigor where it belongs |
| Paraphrasing plan text in flags instead of quoting | 2 | Ctrl+F test: can you find these exact words in the original input? |
| Writing "this needs to be addressed" as the patch | 3 | A patch is "Add X to Y" — implementable without asking what you meant |
| Classifying based on the vision instead of the problem | 1 | Re-read the problem statement only. Ignore the design items. What class is the *problem*? |

## Important

- This skill runs 5 checks and one closing step. That's it. Do not expand scope.
- Do not re-audit the plan. Do not add new flags. Do not restructure the lanes.
- If all 5 pass, output the gate check block, then run Step 6. The gate is not done until the decision is asked.
- Step 6 cannot be satisfied by editing text. Skipping the gate skips the decision, which is the failure the ordering exists to prevent.
- Corrections are surgical — change the minimum text needed to fix the issue.

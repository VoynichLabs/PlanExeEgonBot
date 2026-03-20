# Proposal for Simon: Agent-Executable Plan Output

**Author:** Egon (HejEgonBot)
**Date:** 2026-03-20
**Status:** Ready for review
**For:** PlanExeOrg/PlanExe PR
**Evidence base:** All evidence is inline in this document. Supporting research is in [VoynichLabs/swarm-coordination](https://github.com/VoynichLabs/swarm-coordination) under `plans/egon-dogfood-pr/` and `research/`.

---

## One-Sentence Summary

PlanExe's analytical pipeline is strong; its output format prevents agents from executing the plans it generates. Fix the format, keep the engine.

---

## The Evidence

### What makes plans executable (168 plans, 8 months)

Analysis of 168 dated plan files from the [arc-explainer](https://github.com/markbarney/arc-explainer) project (4,185 commits, Sep 2025 – Mar 2026) identified 8 structural patterns that predict whether a plan ships:

1. **Specific file paths** — plans naming exact files ship at ~85% vs ~30% without (#1 predictor)
2. **Narrow scope (≤80 lines)** — shipped plans avg ~80 lines; 200+ line plans stall or get decomposed
3. **Numbered steps** — agents treat numbered lists as checklists (+40pp ship rate vs bullets)
4. **Root cause before solution** — bug fix plans with diagnosis first ship at +50pp
5. **CHANGELOG forcing function** — plans referenced in CHANGELOG entries ship at ~90% vs ~35%
6. **Explicit non-goals** — prevents scope creep mid-execution
7. **Acceptance criteria** — verifiable done-state
8. **Investigation/implementation split** — prevents stalled audits

PlanExe's current output scores **1.0 out of 8** on these patterns on average. PremiseAttack scores best (2/8 — strong on root cause), WBS L3 tasks score worst (0.5/8 — no file paths, no steps, no acceptance criteria).

### Two structural disconnects

**Disconnect A — Diagnosis doesn't propagate to execution:** PremiseAttack correctly identifies plan failures in Phase 2, but WBS tasks in Phase 23 are generated without those critique findings as grounding constraints. Example from the Batman RICO run: PremiseAttack says "budget is insufficient for forensic accounting" — WBS task 23.7 says "allocate the budget" with zero awareness of the critique.

**Disconnect B — Generation stops where execution begins:** PlanExe outputs task descriptions ("what to do"), not executable procedures. No numbered steps, no file paths, no done-state. An agent receives a correct goal with no verifiable procedure.

### 22 real failures from 47 days of lobster ops

22 confirmed failures across 47 days of three OpenClaw agents (Egon, Bubba, Larry) operating on PlanExe trace to **two root causes**:
- **No prerequisite check before task execution** (config cascades, wrong adapter, stale credentials, missing env vars)
- **No acceptance criterion to verify task output** (wrong metrics shipped, fabricated website content, community contributors in data)

### LODA-Agent DSL feasibility

LODA assembly (Simon's language for integer sequences, [loda-lang.org](https://loda-lang.org/)) is viable as a notation and audit tool for expressing plan structure — 7 of 8 gaps are addressable. The Hybrid Monitor architecture (PlanExe compiles natural language → LODA spec → monitor validates execution) closes both structural disconnects. Registers hold task handle IDs (control plane), not content (data plane). The `lpb` convergence guard enforces bounded execution. 4 dialect extensions are needed: `brk` (early termination), `$0` convention (plan status), `label`/`call`, `par`/`await`.

---

## Three Concrete Asks

### Ask 1: AgentPlanTask (new Luigi task, Phase 24)

**What:** A new task at the end of the pipeline that takes existing WBS + PremiseAttack output and emits an 80-line agent-executable plan.

**Output format** (derived from the 5 features that predict shipping):

```markdown
# Plan: [title from WBS]

## Goal
[one sentence — what done looks like]

## Current State
[what exists now, from PremiseAttack context]

## Non-Goals
[explicitly what this plan does NOT cover — from WBS scope boundaries]

## Prerequisites
[from PremiseAttack findings — encoded as checkable conditions]
- [ ] Budget confirmed ≥ $X (PremiseAttack finding: "budget insufficient")
- [ ] Permit Y obtained (PremiseAttack finding: "regulatory risk")

## Steps
1. [specific action with file path or concrete target]
2. [next action — depends on step 1]
3. ...

## Acceptance Criteria
- [ ] [verifiable condition 1]
- [ ] [verifiable condition 2]

## Completion Record
[filled in after execution — what shipped, what diverged, what was learned]
```

**Why this format:** Plans with file paths ship at ~85% vs ~30% without. Plans under 80 lines ship at ~75% vs ~25% for 200+ lines. Numbered steps correlate with +40pp ship rate vs bullets. These aren't opinions — they're from 168 real plans across 8 months of agent-driven development (see Evidence section above).

**Key design principle:** PremiseAttack findings become Prerequisites, not just a report. The critique *gates* the execution rather than sitting in a separate document.

### Ask 2: LODA-Agent Notation (optional, Phase 25)

**What:** PlanExe compiles the AgentPlanTask output into a LODA-Agent program — a formal specification of the plan's control flow.

**Why LODA specifically:**
- Simon already built LODA and understands the computational model deeply
- The instruction set maps cleanly to agent task coordination (see LODA-Agent DSL feasibility above)
- `lpb` convergence guard enforces bounded execution — no infinite loops
- Integer-only limitation dissolves when registers hold task handle IDs, not content ("control plane / data plane split")
- The miner's mutation vocabulary (`genome.rs`) suggests future automated plan improvement

**What it looks like** (prototype):

```asm
; Plan: Fix PremiseAttack schema validation bug
; Generated by PlanExe AgentPlanTask → LODA-Agent compiler

mov $0,1          ; task: read error log
seq $0,99         ; agent executes → $0 = diagnosis handle
mov $1,$0         ; carry diagnosis forward

geq $1,80         ; GATE: confidence ≥ 80% (from PremiseAttack)
brk $1            ; abort if diagnosis uncertain

mov $0,2          ; task: implement fix
seq $0,99         ; agent executes → $0 = fix handle

mov $0,3          ; task: run tests
seq $0,99         ; agent executes → $0 = test result
equ $0,1          ; ACCEPTANCE: tests pass
brk $0            ; abort if tests fail
```

**Hybrid Monitor:** A lightweight runtime that reads the LODA-Agent program and validates execution against it — checking that gates pass, steps execute in order, and acceptance criteria are met. Not a LODA interpreter — an audit trail validator.

### Ask 3: CompletionRecordTask (post-execution)

**What:** After plan execution, the agent writes a completion record: what shipped, what files changed, what diverged from the plan.

**Why:** The arc-explainer CHANGELOG is a forcing function — entries that reference a plan have a ~90% ship rate vs ~35% for entries without plan references. The completion record closes the loop.

**Format:**

```markdown
## Completion Record — [plan title]
- **Status:** Completed / Partial / Abandoned
- **Date:** YYYY-MM-DD
- **Steps completed:** 1, 2, 3, 5 (step 4 skipped — [reason])
- **Files changed:** [list]
- **Divergences from plan:** [what happened differently and why]
- **Lessons:** [what to do differently next time]
```

This feeds back into future planning — PlanExe can read completion records to calibrate time estimates, identify recurring failure patterns, and improve prerequisite detection.

---

## What This Does NOT Propose

- **Not replacing the 63-task pipeline.** The analytical engine is genuinely valuable. PremiseAttack, Premortem, SWOT, WBS decomposition — all stay. This adds output formatting, not new analysis.
- **Not building a full LODA interpreter.** The Hybrid Monitor validates execution against a spec. It doesn't run agents — OpenClaw does that.
- **Not automating away human oversight.** Prerequisites derived from PremiseAttack create human-approval gates, not bypasses.

---

## Phasing

**Phase 1 (dogfood):** Manually write AgentPlanTask-format plans for 3 real lobster tasks. Execute them. Measure ship rate vs our current ad-hoc approach. Validate the format works before automating it.

**Phase 2 (automate):** Implement AgentPlanTask as a Luigi task. PlanExe generates the 80-line plan from its own WBS + PremiseAttack output. Test on 5 real prompts.

**Phase 3 (LODA-Agent):** Implement the LODA-Agent compiler and Hybrid Monitor. PlanExe output becomes formally verifiable. Test on the same 5 prompts.

**Phase 4 (completion loop):** Implement CompletionRecordTask. Feed completion records back into the planning pipeline. Measure whether plans improve over iterations.

---

## Open Questions for Simon

1. Should AgentPlanTask consume ALL prior task outputs, or only WBS + PremiseAttack? (Leaner = faster, broader = more informed)
2. Is LODA-Agent interesting enough to pursue, or is the AgentPlanTask format sufficient without formal notation?
3. Where does the CompletionRecordTask write its output — same run directory, or a separate feedback repo?
4. Does the `lpb` convergence guard in LODA-Agent match your intuition about how agent execution should be bounded?

---

*This proposal is grounded in 168 real plans from [arc-explainer](https://github.com/markbarney/arc-explainer), 22 real failures from 47 days of lobster ops, 5 prototype LODA-Agent programs, and scored PlanExe output (avg 1.0/8 on the patterns that predict shipping). Supporting research is in [VoynichLabs/swarm-coordination](https://github.com/VoynichLabs/swarm-coordination) under `plans/egon-dogfood-pr/` (analysis files 01–04) and `research/loda-dsl-exploration/` (LODA prototypes and feasibility study).*

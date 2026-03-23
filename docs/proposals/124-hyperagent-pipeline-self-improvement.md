# Proposal: Hyperagent-Style Pipeline Self-Improvement for PlanExe

**Author:** Egon (EgonBot)
**Date:** 2026-03-23
**Status:** Draft — requesting review from neoneye
**Inspired by:** [Hyperagents (arXiv:2603.19461)](https://arxiv.org/abs/2603.19461)
**Related proposals:** #119 (lever pipeline impl), #121 (plan-spawns-agents), #123 (evidence discipline)

---

## The Paper

The Hyperagents paper (Meta / 2026) extends the Darwin Gödel Machine by making the *meta-level improvement procedure itself editable*. Standard DGM improves the task agent; DGM-Hyperagents also improves the mechanism that generates improvements. Key claim: this breaks the domain-alignment constraint that limited DGM to coding (where task performance and self-modification skill are the same skill).

The architecture: a task agent and a meta agent are combined into a **single editable program**. The meta agent can modify both the task agent and itself. Metacognitive self-modification: improving not just what you do, but how you decide what to do next.

---

## The PlanExe Connection

PlanExe already has the structural ingredients for a hyperagent architecture. The question is whether to make that structure explicit and self-improving.

### What we already have

1. **A task pipeline** — the 63-task execution graph that generates plans
2. **A meta-level pipeline** — the lever identification system (#119) that finds which decisions control maximum outcome
3. **A critic pipeline** — PremiseAttack, PremortemTask, the quality gates throughout
4. **A proposal system** — these documents, which propose changes to the pipeline itself
5. **A review loop** — Simon reviewing PRs, accepting or rejecting modifications to the system

The pipeline already modifies itself — just slowly, via PRs, with human review at every step. Hyperagents suggests asking: what if that modification loop were faster, more systematic, and partially automated?

---

## The Proposal

### Core idea: separate the three levels

**Level 1 — Task layer:** The existing 63-task pipeline. Generates plans. We know how to measure its quality (quality gates, downstream human satisfaction, lever pipeline output quality).

**Level 2 — Meta layer:** A new `PipelineOptimizeTask` that analyzes Level 1 output quality and proposes prompt modifications. This is the agent-optimized pipeline idea from #86, formalized as a first-class pipeline stage.

**Level 3 — Meta-meta layer:** A new `ProposalGenerationTask` that analyzes patterns across Level 2 recommendations and generates structured PR proposals — like this one — for human review. This is the editable improvement mechanism.

The key constraint: **Level 3 output requires human review before any Level 1 modification.** This is the safety valve that DGM-Hyperagents lacks in the fully autonomous setting. For PlanExe, human-in-the-loop at the meta-meta level is a feature, not a limitation.

---

### Level 2: PipelineOptimizeTask

A new task that runs after any pipeline completion and analyzes what went wrong or suboptimally:

**Input:**
- Full pipeline run output (all task results)
- Quality gate scores
- Lever pipeline output (if lever tasks ran)
- Prior PipelineOptimizeTask outputs for comparison

**What it does:**
1. Identifies which tasks produced weak output (low confidence, high uncertainty, downstream cascades)
2. Proposes prompt modifications for each weak task
3. Estimates which modifications would most improve overall output quality
4. Flags modifications that are high-risk (could break other tasks)

**Output format:**
```json
{
  "run_id": "...",
  "weak_tasks": [
    {
      "task_name": "IdentifyPotentialLeversTask",
      "failure_mode": "mixing lever types with workstreams",
      "proposed_prompt_delta": "Add explicit ontology classification requirement...",
      "estimated_quality_lift": 0.15,
      "risk_level": "low",
      "prior_proposals": ["Proposal #119 section 3.1 already addresses this"]
    }
  ],
  "top_recommendations": [...],
  "generated_proposal_draft": "..."
}
```

This task runs **every time** — not just when things fail. Continuous improvement signal from every run.

---

### Level 3: ProposalGenerationTask

Aggregates PipelineOptimizeTask outputs across multiple runs and generates a structured proposal document (like this one) for human review:

**Input:**
- PipelineOptimizeTask outputs from N recent runs
- Existing proposals (to avoid duplication)
- Track record of which past proposals improved quality vs. regressed it

**What it does:**
1. Identifies recurring failure patterns across runs
2. Drafts a proposal document in the established format
3. Estimates impact and risk for each proposed change
4. Routes to the right reviewer (Simon for architectural changes, prompt-level changes can go to a faster review loop)

**Key constraint:** ProposalGenerationTask never modifies the pipeline directly. It generates a document that a human reads and decides whether to act on. The improvement mechanism is explicit and auditable.

---

### The lever connection

Proposal #119 identifies levers — the minimal set of decisions controlling maximum outcome. The same insight applies to the pipeline itself: there are pipeline levers (prompt choices, task ordering, quality gate thresholds) that control the quality of PlanExe's output.

A natural extension: after the lever pipeline identifies levers in a *plan*, run `PipelineOptimizeTask` with the lever output as additional context — asking "did the pipeline correctly identify the key levers for this domain?"

This creates a feedback loop:
```
Plan run → Lever pipeline identifies key decisions
→ PipelineOptimizeTask checks if lever pipeline output was correct
→ ProposalGenerationTask proposes lever pipeline improvements
→ Human reviews and merges
→ Better lever identification next run
```

---

## What This Is Not

This is **not** autonomous self-modification. PlanExe should never edit its own prompts at runtime without human review. The DGM-Hyperagents architecture is interesting but its safety properties depend on evaluation environments that PlanExe doesn't have (clear benchmark scores, sandboxed modification). 

What this proposal is: **a structured, auditable improvement loop** — making the current implicit process (someone notices a problem, writes a proposal, Simon reviews it) explicit and partially automated.

---

## Implementation Plan

**Phase 1 (minimal):**
- Add `PipelineOptimizeTask` as an optional post-run analysis task (off by default, enabled with a flag)
- Output: JSON report of weak tasks + draft proposal text
- No pipeline modification, no automation — just better visibility

**Phase 2:**
- Add `ProposalGenerationTask` that aggregates Phase 1 outputs across runs
- Produces formatted proposal documents in `docs/proposals/`
- Still requires human review and manual PR

**Phase 3 (optional, only if Phase 2 proves useful):**
- Fast-path review for low-risk prompt modifications (not architectural changes)
- Track record: measure which proposals actually improved quality when merged
- Use track record to calibrate ProposalGenerationTask's recommendations

---

## Open Questions for Simon

1. Is the pipeline's own improvement loop worth formalizing, or is the current ad-hoc PR process good enough?

2. The Hyperagents paper shows that meta-level improvements transfer across domains — does this hold for PlanExe's prompt modifications? (A better lever-identification prompt might not transfer to a better premise-attack prompt.)

3. Is the right granularity for PipelineOptimizeTask at the prompt level, the task level, or the stage level? The lever pipeline already does something similar for plans — could the same methodology be applied to the pipeline itself?

4. What would a good evaluation metric look like? Quality gate scores exist but they measure output, not pipeline efficiency. What does "this run was better than last run" mean for PlanExe?

---

*This proposal was written in response to neoneye sharing the Hyperagents paper (arXiv:2603.19461) and asking for a PR proposal with it in mind. The core insight — that PlanExe's improvement loop is structurally analogous to DGM's meta-level, and could be made explicit — is Egon's synthesis.*

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

PlanExe already has the structural ingredients for a hyperagent architecture — and has already proven part of it works.

**The `self_improve/` directory (Proposal 117) is `PipelineOptimizeTask` for one task.** It runs `IdentifyPotentialLeversTask` across baseline training data, iterates on the prompt (fix → test → analyze → keep/revert), and produces an auditable trail with 26 iterations completed. The `runner.py` already supports multiple steps via `--step` flag — `identify_potential_levers`, `deduplicate_levers`, `identify_documents`.

Proposal 124 is not starting from scratch. It is proposing to:
1. Generalize what `self_improve/` already does for a few tasks to the full 63-task pipeline
2. Add the accumulation layer — the track record — that makes the improvement loop persistent and learnable

Simon almost certainly had this context when he shared the Hyperagents paper. The question is not "should we build this" — we've already built part of it. The question is "what's the right architecture for the accumulation layer on top of what exists?"

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

## The Accumulation Property (What Makes This Worth Doing)

The Hyperagents paper's key contribution isn't the editable meta-agent — it's that **meta-level improvements transfer across domains and accumulate across runs**. That's what DGM couldn't do. That's what PlanExe's current improvement loop also can't do.

Right now the improvement mechanism is Simon reviewing PRs. It's not editable, it doesn't accumulate, and it doesn't transfer. Each PR is a one-off. The signal from Simon's accept/reject decision disappears.

For this proposal to be worth implementing, `ProposalGenerationTask` needs a **track record** — a structured log of every proposal it generated, whether Simon merged it, and whether quality gate scores improved after the merge. Without that log, it's just generating proposals into the void. With it:

- It learns Simon's review patterns: "architectural changes take longer," "prompt-level lever changes have high acceptance rate," "proposals with before/after examples get merged faster"
- It routes accordingly: architectural changes to Simon with full context; low-risk prompt changes with a faster review signal
- The improvement mechanism gets better at improving — the transfer insight from the paper

The transfer property for PlanExe: a better proposal mechanism that learns Simon's patterns should work whether the plan is about a coffee shop or a biotech startup. The *pipeline structure* is the same even when the domain changes.

**Phase 1 only matters if it's designed to feed the accumulation loop.** Visibility for its own sake isn't the point. The question is: what data format in Phase 1 makes Phase 2 accumulation possible?

**Phase 1 doesn't start cold.** The existing PR history — 308+ merged PRs, timestamped, with Simon's accept/reject signal already baked in — is the seed data for the track record. `ProposalGenerationTask` can bootstrap from `git log` + PR merge history before the first pipeline run. Phase 1 and Phase 3 don't have to be sequential; visibility and calibration can start together.

**The track record must persist on disk.** "Accumulates across runs" means the file must exist between pipeline invocations — not in memory, not per-run state. If it lives in RAM or gets reset per run, you lose the hyperagent property entirely. A flat append-only file in `docs/proposals/track_record.jsonl` is sufficient.

---

## Implementation Plan

**Phase 1 (foundation — accumulation-first design):**
- Pre-populate `docs/proposals/track_record.jsonl` from `git log` + PR merge history (bootstrap from existing signal)
- Add `PipelineOptimizeTask` as an optional post-run analysis task (off by default)
- Output: structured JSON designed to feed the track record, not just a report
  - `task_name`, `failure_mode`, `proposed_prompt_delta`, `estimated_quality_lift`
  - **Critical:** `proposal_id` field — every proposal gets a stable ID for tracking
- Append output to `docs/proposals/track_record.jsonl` — never deleted, append-only

**Phase 2:**
- Add `ProposalGenerationTask` that aggregates Phase 1 outputs
- When Simon merges a proposal: log the merge + pre/post quality gate scores
- `ProposalGenerationTask` reads the track record before generating new proposals
- Produces formatted proposal documents in `docs/proposals/`

**Phase 3:**
- `ProposalGenerationTask` calibrates on acceptance patterns
- Routes by learned signal: high-acceptance proposals fast-tracked, architectural changes flagged for Simon's full review
- Track record becomes a first-class artifact — reviewed periodically like any other pipeline output

---

## Open Questions for Simon

1. **Is the accumulation loop the point?** The Hyperagents paper's real contribution is that meta-improvements accumulate and transfer. If the answer to "does PlanExe need this?" depends on whether Simon's PR review patterns are learnable and repeatable, that's the question to answer first.

2. **Does transfer hold for PlanExe?** The paper shows meta-improvements transfer across domains. Does a better lever-identification prompt transfer to a better premise-attack prompt? Or are PlanExe's tasks too domain-specific for cross-task transfer?

3. **What's the right accumulation granularity?** Track records at the proposal level (did this PR get merged?) or at the quality-gate level (did this prompt change improve scores?) or both?

4. **Phase 1 design question:** What data format in `PipelineOptimizeTask` output makes Phase 2 accumulation actually possible? Getting this wrong in Phase 1 means rebuilding everything in Phase 2.

5. **Is the self_improve/ directory the right home for this?** The repo already has a `self_improve/` directory (spotted in the branch list). Is there existing work there that overlaps with this proposal?

---

*This proposal was written in response to neoneye sharing the Hyperagents paper (arXiv:2603.19461) and asking for a PR proposal with it in mind. The core insight — that PlanExe's improvement loop is structurally analogous to DGM's meta-level, and could be made explicit — is Egon's synthesis.*

---
name: megaplan
description: "Enhanced consensus planning with tri-provider review. Use when user says 'megaplan', 'mega plan', 'tri-provider plan'. Extends standard planning by adding Codex CLI (GPT) and Gemini CLI cross-review after Claude consensus, creating a multi-AI validated plan."
---

# Megaplan -- Tri-Provider Consensus Planning

Enhanced planning skill that adds **Codex CLI (GPT)** and **Gemini CLI** cross-review
to the standard Claude Planner -> Architect -> Critic consensus workflow.

**Result**: A plan validated by 3 independent AI providers (Claude, GPT, Gemini).

## Workflow

```
Phase 1: Claude Consensus (standard planning)
  Planner -> Architect -> Critic -> iterate until agreement

Phase 2: Cross-Provider Review (parallel)
  Codex CLI --+
              +-- review plan in parallel -> collect opinions
  Gemini CLI -+

Phase 3: Synthesis
  Compare all feedback -> identify agreements / disagreements

Phase 4: Resolution (if needed)
  Major disagreements -> refine plan -> re-review
  Minor or none -> proceed to final output

Phase 5: Final Output
  Present consensus plan with tri-provider confidence
```

## Phase 1: Claude Consensus

Run the standard planning flow using Claude agents:

1. **Planner** (opus): Create initial plan from user's task description

   ```
   Task(subagent_type="planner", model="opus", prompt="Create a detailed implementation plan for: {task}")
   ```

2. **Architect** (opus): Review for technical feasibility

   ```
   Task(subagent_type="architect", model="opus", prompt="Review this plan for technical feasibility: {plan}")
   ```

3. **Critic** (opus): Evaluate completeness and quality

   ```
   Task(subagent_type="critic", model="opus", prompt="Critically evaluate this plan: {plan}")
   ```

4. Iterate until all three agree. Save the consensus plan to `plans/megaplan-{timestamp}.md`.

## Phase 2: Cross-Provider Review

Send the Claude-consensus plan to **both** external CLIs **in parallel** using background tasks.

### Codex CLI Review

```bash
codex exec -p "You are a senior software architect reviewing a plan.

## Plan to Review:
{claude_consensus_plan}

## Review Criteria:
1. Technical feasibility -- are there architectural risks or oversights?
2. Completeness -- are any critical steps missing?
3. Dependencies -- are external dependencies properly identified?
4. Timeline realism -- is the sequencing reasonable?
5. Risk assessment -- what could go wrong?

Provide your review as:
- VERDICT: APPROVE / APPROVE_WITH_CONCERNS / REJECT
- STRENGTHS: (bullet list)
- CONCERNS: (bullet list with severity: HIGH/MEDIUM/LOW)
- SUGGESTIONS: (bullet list)
- MISSING: (anything not covered)

Be thorough but concise." 2>&1 > /tmp/megaplan_codex_review.txt
```

### Gemini CLI Review

```bash
gemini -m gemini-2.5-pro -p "You are a senior software architect reviewing a plan.

## Plan to Review:
{claude_consensus_plan}

## Review Criteria:
1. Technical feasibility -- are there architectural risks or oversights?
2. Completeness -- are any critical steps missing?
3. Dependencies -- are external dependencies properly identified?
4. Timeline realism -- is the sequencing reasonable?
5. Risk assessment -- what could go wrong?

Provide your review as:
- VERDICT: APPROVE / APPROVE_WITH_CONCERNS / REJECT
- STRENGTHS: (bullet list)
- CONCERNS: (bullet list with severity: HIGH/MEDIUM/LOW)
- SUGGESTIONS: (bullet list)
- MISSING: (anything not covered)

Be thorough but concise." 2>&1 > /tmp/megaplan_gemini_review.txt
```

**IMPORTANT**: Run both in parallel using `run_in_background: true` on each Bash call.

## Phase 3: Synthesis

After both reviews complete, synthesize the results:

1. Read both review files (`/tmp/megaplan_codex_review.txt`, `/tmp/megaplan_gemini_review.txt`)
2. Create a synthesis table:

```markdown
## Tri-Provider Consensus Report

| Aspect        | Claude (Planner+Architect+Critic) | Codex (GPT) | Gemini    |
| ------------- | --------------------------------- | ----------- | --------- |
| Verdict       | {verdict}                         | {verdict}   | {verdict} |
| Key Strengths | ...                               | ...         | ...       |
| Key Concerns  | ...                               | ...         | ...       |

### Agreements (all 3 providers agree)

- ...

### Disagreements (providers differ)

- ...

### Combined Suggestions (deduplicated)

- ...
```

## Phase 4: Resolution

Based on the synthesis:

- **All APPROVE**: Proceed to Phase 5 (final output)
- **APPROVE_WITH_CONCERNS**: Address HIGH-severity concerns, then proceed
- **Any REJECT**: Identify rejection reasons, refine the plan, re-run Phase 2
- **Max 2 iterations** of Phase 2-4 to prevent infinite loops

When refining:

- Feed all three reviews back to the Planner agent
- Ask Planner to address specific concerns raised by Codex/Gemini
- Re-run Architect + Critic on the revised plan
- Re-run Codex + Gemini review on the revised plan

## Phase 5: Final Output

Present the final plan with confidence metrics:

```markdown
# Megaplan Result -- {task_title}

## Consensus Status

- Claude: {verdict}
- Codex (GPT): {verdict}
- Gemini: {verdict}
- **Overall**: {CONSENSUS_REACHED / MAJORITY_APPROVE / NO_CONSENSUS}
- **Iterations**: {n}

## Final Plan

{the_plan}

## Review Summary

{synthesis_table_from_phase_3}

## Action Items

{prioritized_list_from_all_reviews}
```

Save the final output to `plans/megaplan-{timestamp}-final.md`.

## Error Handling

- **Codex CLI fails**: Log error, proceed with Claude + Gemini only (2-provider consensus)
- **Gemini CLI fails (503)**: Retry once with 30s delay. If still fails, proceed with Claude + Codex only
- **Both CLIs fail**: Fall back to standard planning (Claude-only consensus) and notify user
- **Timeout**: 3 minutes per CLI review. Kill and proceed if exceeded.

## Configuration

Environment requirements:

- `codex` CLI installed and authenticated (OpenAI account)
- `gemini` CLI installed with `GEMINI_API_KEY` set
- Use a specific Gemini model flag to avoid 503 errors on default model

## Tips

- For **code-heavy plans**, Codex (GPT) tends to give stronger technical feedback
- For **design/UX plans**, Gemini tends to give stronger structural feedback
- For **architecture plans**, Claude Architect tends to give the deepest analysis
- The tri-provider approach catches blind spots that any single AI would miss

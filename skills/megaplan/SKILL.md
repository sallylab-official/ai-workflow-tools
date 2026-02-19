---
name: megaplan
description: "Enhanced consensus planning with tri-provider review. Use when user says 'megaplan', 'mega plan', 'tri-provider plan', '3자 검토', '메가플랜'. Extends ralplan by adding Codex CLI (GPT) and Gemini API cross-review after Claude consensus, creating a multi-AI validated plan."
---

# Megaplan — Tri-Provider Consensus Planning

Enhanced version of `ralplan` that adds **Codex CLI (GPT-5.2)** and **Gemini API (gemini-2.0-flash + web search)** cross-review
to the standard Claude Planner → Architect → Critic consensus workflow.

**Result**: A plan validated by 3 independent AI providers (Claude, GPT, Gemini).

## Workflow

```
Phase 1: Claude Consensus (standard ralplan)
  Planner → Architect → Critic → iterate until agreement

Phase 2: Cross-Provider Review (parallel)
  Codex CLI (GPT-5.2) ──┐
                         ├── review plan in parallel → collect opinions
  Gemini API (2.0-flash) ┘

Phase 3: Synthesis
  Compare all feedback → identify agreements / disagreements

Phase 4: Resolution (if needed)
  Major disagreements → refine plan → re-review
  Minor or none → proceed to final output

Phase 5: Final Output
  Present consensus plan with tri-provider confidence
```

## Phase 1: Claude Consensus

Run the standard ralplan flow using Claude agents:

1. **Planner** (opus): Create initial plan from user's task description

   ```
   Task(subagent_type="oh-my-claudecode:planner", model="opus", prompt="Create a detailed implementation plan for: {task}")
   ```

2. **Architect** (opus): Review for technical feasibility

   ```
   Task(subagent_type="oh-my-claudecode:architect", model="opus", prompt="Review this plan for technical feasibility: {plan}")
   ```

3. **Critic** (opus): Evaluate completeness and quality

   ```
   Task(subagent_type="oh-my-claudecode:critic", model="opus", prompt="Critically evaluate this plan: {plan}")
   ```

4. Iterate until all three agree. Save the consensus plan to `.omc/plans/megaplan-{timestamp}.md`.

## Phase 2: Cross-Provider Review

**STEP 1**: Write the plan + review prompt to `/tmp/megaplan_prompt.txt` first.
This avoids shell escaping issues with special characters.

```bash
cat > /tmp/megaplan_prompt.txt << 'PROMPT_EOF'
You are a senior software architect reviewing a plan.

## Plan to Review:
{claude_consensus_plan}

## Review Criteria:
1. Technical feasibility — are there architectural risks or oversights?
2. Completeness — are any critical steps missing?
3. Dependencies — are external dependencies properly identified?
4. Timeline realism — is the sequencing reasonable?
5. Risk assessment — what could go wrong?

Provide your review as:
- VERDICT: APPROVE / APPROVE_WITH_CONCERNS / REJECT
- STRENGTHS: (bullet list)
- CONCERNS: (bullet list with severity: HIGH/MEDIUM/LOW)
- SUGGESTIONS: (bullet list)
- MISSING: (anything not covered)

Be thorough but concise. Korean or English is fine.
PROMPT_EOF
```

**STEP 2**: Run both reviews in parallel using `run_in_background: true`.

### Codex CLI Review (GPT-5.2, with built-in web search)

```bash
codex exec --skip-git-repo-check "$(cat /tmp/megaplan_prompt.txt)" 2>&1 > /tmp/megaplan_codex_review.txt
echo "Codex done: $?"
```

Key flags:

- `--skip-git-repo-check`: **Required** when running outside a git repository
- Default model is `gpt-5.2` (xhigh reasoning) — do NOT override
- Codex has built-in web search — no extra configuration needed

### Gemini API Review (gemini-2.0-flash + Google Search grounding)

Use Python API directly — **do NOT use the `gemini` CLI** (it has MCP server conflicts and wrong model names).

```bash
source ~/.zshrc && python3 - << 'EOF' > /tmp/megaplan_gemini_review.txt 2>&1
import json, urllib.request, os

prompt = open("/tmp/megaplan_prompt.txt").read()
api_key = os.environ.get("GEMINI_API_KEY")
if not api_key:
    print("ERROR: GEMINI_API_KEY not set in environment")
    exit(1)

url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent?key={api_key}"
payload = json.dumps({
    "contents": [{"parts": [{"text": prompt}]}],
    "tools": [{"google_search": {}}]
}).encode()

req = urllib.request.Request(url, data=payload, headers={"Content-Type": "application/json"})
try:
    with urllib.request.urlopen(req, timeout=180) as r:
        data = json.load(r)
    for part in data["candidates"][0]["content"]["parts"]:
        if "text" in part:
            print(part["text"])
except Exception as e:
    print(f"ERROR: {e}")
    exit(1)
EOF
echo "Gemini done: $?"
```

Notes:

- `google_search` tool enables **real-time web research** during the review
- Model: `gemini-2.5-pro` — highest reasoning, supports search grounding
- GEMINI_API_KEY must be in `~/.zshrc` (loaded via `source ~/.zshrc`)

## Phase 3: Synthesis

After both background tasks complete (use `TaskOutput` to wait), read both files and synthesize:

```markdown
## Tri-Provider Consensus Report

| Aspect        | Claude (Planner+Architect+Critic) | Codex (GPT-5.2) | Gemini (2.0-flash) |
| ------------- | --------------------------------- | --------------- | ------------------ |
| Verdict       | {verdict}                         | {verdict}       | {verdict}          |
| Key Strengths | ...                               | ...             | ...                |
| Key Concerns  | ...                               | ...             | ...                |

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

## Phase 5: Final Output

```markdown
# Megaplan Result — {task_title}

## Consensus Status

- Claude: {verdict}
- Codex (GPT-5.2): {verdict}
- Gemini (2.0-flash + web): {verdict}
- **Overall**: {CONSENSUS_REACHED / MAJORITY_APPROVE / NO_CONSENSUS}
- **Iterations**: {n}

## Final Plan

{the_plan}

## Review Summary

{synthesis_table_from_phase_3}

## Action Items

{prioritized_list_from_all_reviews}
```

## Error Handling

- **Codex "Not inside a trusted directory"**: Add `--skip-git-repo-check` flag (already in template)
- **Codex fails entirely**: Log error, proceed with Claude + Gemini only
- **Gemini API 404**: Model name wrong — only use `gemini-2.0-flash` (not `gemini-3-pro-preview`)
- **Gemini API auth error**: `GEMINI_API_KEY` not loaded — run `source ~/.zshrc` first
- **Both fail**: Fall back to Claude-only consensus and notify user
- **Timeout**: 3 minutes per review. If exceeded, kill and proceed with available results.

## Configuration

Requirements:

- `codex` CLI installed and authenticated (ChatGPT Plus account)
- `GEMINI_API_KEY` set in `~/.zshrc`
- Always run `source ~/.zshrc` before the Gemini Python call

## Model Reference

| Provider | Model             | Reasoning | Web Search                        |
| -------- | ----------------- | --------- | --------------------------------- |
| Claude   | claude-opus-4-6   | Highest   | No (use WebFetch/WebSearch tools) |
| Codex    | gpt-5.2 (default) | xhigh     | Built-in                          |
| Gemini   | gemini-2.5-pro    | Highest   | `google_search` tool              |

## Tips

- For **code-heavy plans**, Codex (GPT-5.2) gives strongest technical feedback
- For **design/strategy plans**, Gemini with web search can pull current best practices
- For **architecture plans**, Claude Architect gives the deepest structural analysis
- The tri-provider approach catches blind spots that any single AI would miss
- **Non-code tasks** (content, strategy, copy): megaplan works for these too — just adapt the review prompt

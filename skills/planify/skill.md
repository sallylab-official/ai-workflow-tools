---
name: planify
description: "Explore codebase → write plan → multi-AI review → Opus approval loop → final complete plan with test plan. Call as /planify [requirement]."
---

# Planify — Requirements → Verified Complete Plan

Takes any requirement and automatically runs: codebase exploration → plan writing → multi-AI review → Opus approval loop.
The final plan always includes a **test plan**.

## Usage

```
/planify [requirement]
```

Examples:

```
/planify Add retry button when image upload fails
/planify Show tooltip explaining why a disabled option is unavailable
```

## Full Workflow

```
Phase 1: Explore (Explore subagent)
  → Identify relevant code, files, and patterns

Phase 2: Plan writing (following writing-plans conventions)
  → Save to docs/plans/YYYY-MM-DD-{feature-name}.md
  → Must include a test plan section

Phase 3: Multi-AI review (ask-ai)
  → Claude Opus + Codex (GPT) parallel opinions
  → Incorporate feedback and update plan

Phase 4: Opus final approval (infinite loop)
  → Run opus-review
  → Revise until "no issues" — repeat if needed
  → Max 5 iterations (then ask user to decide)

Phase 5: Summary report
  → File path, review rounds, key decisions
```

---

## Phase 1: Explore

Use an Explore subagent to find code relevant to the requirement.

**Explore subagent prompt:**

```
Project root: {cwd}
Requirement: {arg1}

Find and document:
1. Relevant file paths + key line numbers
2. Current implementation (code patterns, data flow)
3. Architecture components involved (state management, services, repositories)
4. Similar existing features (for pattern reference)
5. Other files that will be affected by changes

Include specific file paths and line numbers in results.
```

---

## Phase 2: Plan Writing

### Required structure

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** [one sentence]

**Architecture:** [2-3 sentence approach]

**Tech Stack:** [key technologies/libraries]

---

### Task 1: ...

### Task 2: ...

...

---

## Test Plan ← required

### Manual Test Checklist

| #   | Scenario | Expected Result | Check |
| --- | -------- | --------------- | ----- |
| 1   | ...      | ...             | [ ]   |
| 2   | ...      | ...             | [ ]   |

### Edge Cases

- [ ] ...
- [ ] ...

### Regression Tests (verify existing features unaffected)

- [ ] ...
```

### Save path

```
docs/plans/YYYY-MM-DD-{feature-name}.md
```

Use today's date.

---

## Phase 3: Multi-AI Review (ask-ai)

### Pre-check

```bash
which codex 2>/dev/null && echo "OK" || echo "not installed"
```

If not installed: proceed with Opus-only review (Phase 4 only).

### Write prompt file

```bash
cat > /tmp/planify_ask_prompt.txt << 'EOF'
Project: {brief project description}
Requirement: {arg1}

[Full plan]
{plan_content}

Review:
1. Does this approach follow the project's existing patterns?
2. Are there any missing features or potential bugs?
3. Is there a simpler implementation?
4. Is the test plan sufficient?

Be concise.
EOF
```

### Run in parallel

```python
# Claude Opus (Task tool)
Task(subagent_type="general-purpose", model="opus", run_in_background=True,
     prompt="<prompt file content>")

# Codex (Bash tool)
Bash("codex \"$(cat /tmp/planify_ask_prompt.txt)\" 2>&1 | tail -40",
     run_in_background=True, timeout=90000)
```

### Incorporate results

Summarize both opinions in a table, then update the plan:

```markdown
| Item | Claude Opus | Codex (GPT) |
| ---- | ----------- | ----------- |
| ...  | ...         | ...         |

**Incorporated**: ...
**Not incorporated + reason**: ...
```

---

## Phase 4: Opus Final Approval (infinite loop)

### Loop

```
iteration = 1
while true:
  1. Summarize current plan in ~300 chars
  2. Opus subagent reviews the plan:
     Prompt:
       [Project] Current codebase context.
       [Full plan] {plan_content}
       [Previously addressed issues] {list of previous issues and whether they were fixed}
       Review: 1) direction 2) potential bugs 3) simpler alternatives 4) pattern violations
       If no issues, explicitly state "no issues".
  3. "no issues" → exit loop
  4. Issues found → revise plan → iteration++
  5. iteration > 5 → ask user to decide, then exit
```

### Opus subagent config

```
subagent_type: general-purpose
model: opus
```

---

## Phase 5: Summary Report

```markdown
## Planify Complete

**Plan file**: `docs/plans/{filename}.md`
**Review rounds**: {n} (ask-ai: 1 round + Opus: {m} rounds)

**Key design decisions**:

- ...

**Test plan**: {n} scenarios, {m} edge cases

**Next step**: Use `superpowers:executing-plans` to start implementation
```

---

## Plan Quality Checklist

Only mark complete when all of these are satisfied:

- [ ] All affected files listed with exact paths
- [ ] Each task includes runnable code snippets
- [ ] Linter command included (e.g., `eslint`, `dart analyze`, `ruff`, `tsc`)
- [ ] Manual device/browser verification checklist included
- [ ] Commit command included
- [ ] **Test plan section exists** ← add if missing

---

## Notes

- Phase 4 repeats until Opus explicitly states "no issues". Do not exit early.
- After revising the plan, always save the file before passing it to Opus again.
- If Codex is not installed, skip Phase 3 and go directly to Phase 4.
- If arg1 is empty, ask the user for the requirement before proceeding.

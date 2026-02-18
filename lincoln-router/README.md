# Lincoln Router -- Prompt Quality Gate

A Claude Code `UserPromptSubmit` hook that evaluates prompt clarity before Claude starts working and routes requests to the appropriate workflow.

## The Problem

Vague prompts produce vague code. When a user says "fix the bug" or "make it work," Claude has to guess at scope, intent, and constraints. The result: wasted iterations, wrong assumptions, and rework.

## The Solution

Lincoln intercepts every user prompt and runs it through a quality gate:

```
User prompt
    |
    v
[Clarity Scoring] -- 0.0 to 1.0 scale
    |
    |-- score < 0.45 ("critical") --> Ask clarifying questions
    |-- score < 0.65 ("unclear")  --> Ask clarifying questions
    |-- score >= 0.65 ("clear")   --> Continue
    |
    v
[Domain Classification]
    |-- coding (implementation, debugging, testing, etc.)
    |-- non_coding (planning, documentation, translation, etc.)
    |
    v
[Context Check] -- Does this request need repo context?
    |-- yes --> Auto-explore (ls, git status, file structure)
    |-- no  --> Proceed directly
    |
    v
[Handoff to normal Claude execution]
```

## Architecture

```
lincoln/
  request_router_stage0.py    # Core routing logic
  conductor_hook/
    lincoln_hook.py           # Claude Code hook entrypoint
    context_builder.py        # Builds context injected into session
    state_manager.py          # Tracks state across turns
```

### Clarity Scoring (`estimate_clarity_score`)

The scorer starts at 1.0 and adjusts based on signals:

| Signal                                 | Effect                         |
| -------------------------------------- | ------------------------------ |
| Empty or very short prompt             | -0.25 to -0.75                 |
| Vague/urgent language patterns         | -0.35                          |
| Conflicting constraints                | -0.30                          |
| Multiple vague words                   | -0.15                          |
| File/path references detected          | +0.15                          |
| Scope/goal keywords present            | +0.10                          |
| Previous clarifying questions answered | +0.10 per question (max +0.25) |

### Domain Classification

Keyword-based classification into `coding`, `non_coding`, or `unknown`. Used to determine whether repo context exploration is needed.

### Question Generation

When clarity is low, Lincoln generates targeted clarifying questions (max 3 per request):

1. "What is the exact end goal in one sentence?"
2. "What is the scope? (folder path, feature name, file name)"
3. "Are there constraints? (quality standards, time, testing requirements)"

The question count is tracked across turns via `StateManager` so the same request doesn't get asked more than 3 times.

### Context Exploration

For coding requests that are clear but lack a specific target, Lincoln triggers an auto-explore phase:

```bash
ls -la
rg --files | head -n 80
git status --short
find . -maxdepth 2 -type f | head -n 40
git log --oneline -n 5
```

This gives Claude the repo context it needs before starting work.

## How It Hooks Into Claude Code

Lincoln uses the `UserPromptSubmit` hook event. It reads the user's prompt from stdin as JSON, processes it, and returns `additionalContext` that gets injected into Claude's session:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "... routing instructions ..."
  }
}
```

## Configuration (settings.json)

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/lincoln/lincoln_hook.py",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

## Building Your Own

This is open-sourced as a design reference. To adapt it for your team:

1. **Replace the keyword lists** -- The current implementation uses Korean keywords for domain classification. Replace with your language/domain terms.
2. **Tune the clarity thresholds** -- 0.45/0.65 work for our team. Adjust based on your prompt patterns.
3. **Customize question templates** -- Write questions that match your workflow.
4. **Add domain-specific signals** -- If your team works in a specific domain, add keyword patterns that detect relevant concepts.

The architecture (score -> classify -> route) is language-agnostic. The specific patterns are not.

## Requirements

- Python 3.8+
- Claude Code with hook support

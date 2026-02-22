# ai-workflow-tools

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> Battle-tested AI workflow tools from a 3-person startup that shipped 4 products in 2.5 months

These are the actual skills and commands we use every day at [SallyLab](https://sallylab.io). Not theoretical frameworks -- real tools extracted from a production workflow that built and shipped:

- **Sally App** -- Flutter mobile app
- **Campaign Web** -- Next.js marketing platform ([campaigns.sallylab.io](https://campaigns.sallylab.io))
- **Admin Dashboard** -- Internal ops tool
- **Partner Portal** -- B2B partner interface

All built by a 3-person team running Claude Code, Gemini CLI, and Codex CLI in tandem.

---

## Tools

| Tool                                  | Type     | What It Does                                            |
| ------------------------------------- | -------- | ------------------------------------------------------- |
| [Megaplan](#megaplan)                 | Skill    | Tri-provider consensus planning (Claude + GPT + Gemini) |
| [Planify](#planify)                   | Skill    | Requirements → verified plan with Opus approval loop    |
| [Handoff Commands](#handoff-commands) | Commands | AI-to-AI session continuity protocol                    |
| [Lincoln Router](#lincoln-router)     | Hook     | Prompt quality gate and workflow router                 |
| [Completion Sound](#completion-sound) | Hook     | Audio feedback when Claude finishes a task              |

---

## Megaplan

**Type**: Claude Code Skill (`~/.claude/skills/megaplan/`)

**Based on**: [oh-my-claude-code](https://github.com/anthropics/claude-code)'s `ralplan` consensus flow (Planner → Architect → Critic)

**What we added and why**: `ralplan` uses Claude-only consensus. In practice, we found that Claude agents reviewing each other's plans share the same blind spots -- they agree too easily. When a wrong architectural decision passes through all three Claude roles, the entire sprint gets derailed.

We extended `ralplan` by adding **Codex CLI (GPT)** and **Gemini CLI** as independent cross-reviewers after the Claude consensus phase:

```
Phase 1: Claude Consensus (standard ralplan)
  Planner --> Architect --> Critic --> iterate until agreement

Phase 2: Cross-Provider Review (parallel)   ← WE ADDED THIS
  Codex CLI (GPT) ---+
                     +--- review plan --> collect opinions
  Gemini CLI --------+

Phase 3: Synthesis                           ← WE ADDED THIS
  Compare all feedback --> agreements / disagreements

Phase 4: Resolution (if any REJECT)          ← WE ADDED THIS
  Refine plan --> re-review (max 2 iterations)

Phase 5: Final Output
  Consensus plan with tri-provider confidence scores
```

**Why it matters**: Each provider has different blind spots. Claude excels at deep architecture analysis, GPT gives stronger technical feedback on code-heavy plans, and Gemini catches structural issues. In our experience, the tri-provider approach caught 2-3 critical issues per plan that Claude-only consensus missed.

**Requirements**: Claude Code, [Codex CLI](https://github.com/openai/codex), [Gemini CLI](https://github.com/google-gemini/gemini-cli)

---

## Planify

**Type**: Claude Code Skill (`~/.claude/skills/planify/`)

**What it does**: Takes a requirement in plain language and drives Claude through a structured 5-phase process: codebase exploration → plan writing → multi-AI review → Opus approval loop → final report. Every plan produced includes a test checklist.

**Why we built it**: Writing plans ad-hoc leads to inconsistent quality. Some plans have test coverage, some don't. Some get reviewed, some go straight to implementation. Planify enforces the full process every time, so every feature starts from a verified, reviewed plan.

```
Phase 1: Explore (Explore subagent)
  → Map relevant files, patterns, and architecture

Phase 2: Write plan
  → docs/plans/YYYY-MM-DD-{feature}.md with test plan section

Phase 3: Multi-AI review (parallel)       ← catches blind spots
  Claude Opus ---+
                 +--- review → synthesize → update plan
  Codex (GPT) --+

Phase 4: Opus approval loop               ← repeats until clean
  Review → revise → re-review (max 5 iterations)

Phase 5: Report
  → File path, round count, key decisions
```

**The key insight**: Opus reviewing its own plans finds issues that in-context Claude misses. The approval loop (not one-shot review) is what makes the difference -- it forces the plan to be actually fixed, not just flagged.

**Requirements**: Claude Code. [Codex CLI](https://github.com/openai/codex) optional (Phase 3 skipped if not installed).

---

## Handoff Commands

**Type**: Claude Code Custom Commands (`~/.claude/commands/`)

The biggest pain point with AI coding agents is **context loss between sessions**. You spend 30 minutes getting Claude up to speed, it does great work, the session ends -- and the next session starts from zero.

We built a structured handoff protocol to solve this:

### `/handoff-create`

Generates a comprehensive `HANDOFF.md` capturing everything the next session needs: goals, completed work, failed approaches (critical -- saves hours of repeating mistakes), key decisions, current state, and step-by-step resume instructions.

### `/handoff-quick`

Minimal version for simple tasks. Goal, done, next step, warnings. Four lines.

### `/handoff-resume`

Reads a `HANDOFF.md`, verifies the repo state hasn't drifted, summarizes context, and picks up exactly where the last session left off. Pays special attention to the "Failed Approaches" section so it doesn't repeat mistakes.

**The key insight**: Failed approaches are mandatory in every handoff. Knowing what _didn't_ work is more valuable than knowing what did.

---

## Lincoln Router

**Type**: Claude Code `UserPromptSubmit` Hook (Python)

Lincoln is a prompt quality gate that evaluates every user request _before_ Claude starts working. It scores prompt clarity on a 0-1 scale and routes to the appropriate workflow:

```
User prompt
    |
    v
[Clarity Scoring] -- score < 0.45 --> "critical" --> Ask clarifying questions
    |                  score < 0.65 --> "unclear"  --> Ask clarifying questions
    v
[Domain Classification] -- coding vs. non-coding
    |
    v
[Context Check] -- needs repo context? --> Auto-explore (ls, git status, etc.)
    |
    v
[Handoff to Stage 1] -- prompt is clear, proceed normally
```

**Why we built it**: Vague prompts produce vague code. We were losing 20-30 minutes per session on misdirected work from ambiguous requests. Lincoln catches ambiguity upfront and asks targeted questions. The result: fewer wasted iterations, more precise output.

**Architecture**:

- `request_router_stage0.py` -- Clarity scoring, domain classification, question generation
- `conductor_hook/lincoln_hook.py` -- Claude Code hook entrypoint, state management
- `conductor_hook/context_builder.py` -- Builds additional context injected into the session
- `conductor_hook/state_manager.py` -- Tracks question count across turns to avoid over-asking

The clarity scorer uses keyword matching for domain classification (coding vs. non-coding), regex patterns for detecting vague/urgent language, and file/path signal detection. It caps clarifying questions at 3 per request to avoid annoying the user.

> **Note**: The full implementation contains workspace-specific paths and Korean NLP patterns tuned to our team's workflow. This is open-sourced as a design reference -- adapt the architecture to your language and patterns.

---

## Installation

### Commands

```bash
cp commands/handoff-create.md ~/.claude/commands/
cp commands/handoff-quick.md ~/.claude/commands/
cp commands/handoff-resume.md ~/.claude/commands/
```

### Skills

```bash
cp -r skills/megaplan ~/.claude/skills/
cp -r skills/planify ~/.claude/skills/
```

### Completion Sound

```bash
mkdir -p ~/.claude/sounds
cp sounds/reze-boom.wav ~/.claude/sounds/
```

Then add the Stop hook to `~/.claude/settings.json` — see [`sounds/README.md`](sounds/README.md) for the full config and platform-specific commands.

### Lincoln Router

See [`lincoln-router/README.md`](lincoln-router/README.md) for architecture details. The implementation requires customization for your team's language and workflow patterns.

---

## Completion Sound

**Type**: Claude Code `Stop` Hook

Plays a sound the moment Claude Code finishes a task. Simple, but surprisingly effective for long-running work -- you can leave your desk and know exactly when to come back.

**How it works**: Claude Code fires a `Stop` event at the end of every response. The hook runs a one-liner that plays a WAV file in the background.

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "[ -f ~/.claude/sounds/reze-boom.wav ] && afplay --volume 1.0 ~/.claude/sounds/reze-boom.wav 2>/dev/null &"
          }
        ]
      }
    ]
  }
}
```

**Install**: Copy `sounds/reze-boom.wav` to `~/.claude/sounds/`, add the hook. Two steps. See [`sounds/README.md`](sounds/README.md) for Linux and Windows (WSL) commands.

---

## Built With

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) -- Primary AI coding agent
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) -- Cross-provider review in Megaplan
- [Codex CLI](https://github.com/openai/codex) -- Cross-provider review in Megaplan
- [oh-my-claude-code](https://github.com/anthropics/claude-code) -- `ralplan` consensus flow (base for Megaplan)

---

## About SallyLab

SallyLab is a 3-person team building AI-powered products. We believe the future of software development is human-AI collaboration, and we build tools to make that collaboration better.

- **Website**: [sallylab.io](https://sallylab.io)
- **Campaigns**: [campaigns.sallylab.io](https://campaigns.sallylab.io)

---

## License

MIT -- see [LICENSE](LICENSE) for details.

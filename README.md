# ai-workflow-tools

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> Battle-tested AI workflow tools from a 3-person startup that shipped 4 products in 2.5 months

These are the actual hooks, skills, and commands we use every day at [SallyLab](https://sallylab.io). Not theoretical frameworks -- real tools extracted from a production workflow that built and shipped:

- **Sally App** -- Flutter mobile app
- **Campaign Web** -- Next.js marketing platform ([campaigns.sallylab.io](https://campaigns.sallylab.io))
- **Admin Dashboard** -- Internal ops tool
- **Partner Portal** -- B2B partner interface

All built by a 3-person team running Claude Code, Gemini CLI, and Codex CLI in tandem.

---

## Tools

| Tool                                  | Type     | What It Does                                              |
| ------------------------------------- | -------- | --------------------------------------------------------- |
| [Megaplan](#megaplan)                 | Skill    | Tri-provider consensus planning (Claude + GPT + Gemini)   |
| [Handoff Commands](#handoff-commands) | Commands | AI-to-AI session continuity protocol                      |
| [RTK Rewrite](#rtk-rewrite-hook)      | Hook     | Transparent CLI command optimization, saves 60-90% tokens |
| [Lincoln Router](#lincoln-router)     | Hook     | Prompt quality gate and workflow router                   |
| [Auto-Prettier](#auto-prettier-hook)  | Hook     | Auto-formats every file Claude touches                    |

---

## Megaplan

**Type**: Claude Code Skill (`~/.claude/skills/megaplan/`)

When the stakes are high, one AI isn't enough. Megaplan runs a structured consensus workflow across three independent AI providers:

```
Phase 1: Claude Consensus
  Planner --> Architect --> Critic --> iterate until agreement

Phase 2: Cross-Provider Review (parallel)
  Codex CLI (GPT) ---+
                     +--- review plan --> collect opinions
  Gemini CLI --------+

Phase 3: Synthesis
  Compare all feedback --> identify agreements / disagreements

Phase 4: Resolution
  Major disagreements --> refine --> re-review (max 2 iterations)

Phase 5: Final Output
  Consensus plan with tri-provider confidence scores
```

**Why it works**: Each provider has different blind spots. Claude excels at deep architecture analysis, GPT gives strong technical feedback on code-heavy plans, and Gemini catches structural issues. The tri-provider approach surfaces problems that any single AI would miss.

**Requirements**: Claude Code, [Codex CLI](https://github.com/openai/codex), [Gemini CLI](https://github.com/google-gemini/gemini-cli)

---

## Handoff Commands

**Type**: Claude Code Custom Commands (`~/.claude/commands/`)

The biggest pain point with AI coding agents is **context loss between sessions**. You spend 30 minutes getting Claude up to speed, it does great work, the session ends -- and the next session starts from zero.

Handoff Commands solve this with a structured protocol:

### `/handoff-create`

Generates a comprehensive `HANDOFF.md` capturing everything the next session needs: goals, completed work, failed approaches (critical -- saves hours of repeating mistakes), key decisions, current state, and step-by-step resume instructions.

### `/handoff-quick`

Minimal version for simple tasks. Goal, done, next step, warnings. Four lines.

### `/handoff-resume`

Reads a `HANDOFF.md`, verifies the repo state hasn't drifted, summarizes context, and picks up exactly where the last session left off. Pays special attention to the "Failed Approaches" section so it doesn't repeat mistakes.

**The key insight**: Failed approaches are mandatory in every handoff. Knowing what _didn't_ work is more valuable than knowing what did.

---

## RTK Rewrite Hook

**Type**: Claude Code `PreToolUse` Hook (`~/.claude/hooks/`)

Claude Code burns tokens on verbose CLI output. `git status` on a large repo? Thousands of tokens. `git log`? Even more.

The RTK Rewrite hook transparently intercepts CLI commands and rewrites them to use [`rtk`](https://github.com/QuantGeekDev/rtk) (Reduced Token Kit) equivalents. Claude never knows the difference -- it writes `git status`, but `rtk git status` actually runs, returning compressed output that saves 60-90% of tokens.

**Supported commands**:

- Git (`status`, `diff`, `log`, `add`, `commit`, `push`, `pull`, `branch`, `fetch`, `stash`, `show`)
- GitHub CLI (`pr`, `issue`, `run`)
- JS/TS (`vitest`, `tsc`, `eslint`, `prettier`, `playwright`, `prisma`)
- Python (`pytest`, `ruff`, `pip`)
- Go (`test`, `build`, `vet`, `golangci-lint`)
- Rust (`cargo test`, `cargo build`, `cargo clippy`)
- Containers (`docker ps/images/logs`, `kubectl get/logs`)
- File ops (`cat` -> `rtk read`, `grep`/`rg`, `ls`)
- Network (`curl`)

**How it works**: The hook reads Claude's Bash tool input via stdin, pattern-matches the command, rewrites it with the `rtk` prefix, and returns a JSON `updatedInput` that Claude Code applies transparently. If `rtk` or `jq` aren't installed, it silently passes through.

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

**Why it matters**: Vague prompts produce vague code. Instead of letting Claude guess, Lincoln catches ambiguity upfront and asks targeted questions. The result: fewer wasted iterations, more precise output.

**Architecture**:

- `request_router_stage0.py` -- Clarity scoring, domain classification, question generation
- `conductor_hook/lincoln_hook.py` -- Claude Code hook entrypoint, state management
- `conductor_hook/context_builder.py` -- Builds additional context injected into the session
- `conductor_hook/state_manager.py` -- Tracks question count across turns to avoid over-asking

The clarity scorer uses keyword matching for domain classification (coding vs. non-coding), regex patterns for detecting vague/urgent language, and file/path signal detection. It caps clarifying questions at 3 per request to avoid annoying the user.

> **Note**: The full implementation contains workspace-specific paths and Korean NLP patterns tuned to our team's workflow. This is open-sourced as a design reference -- adapt the architecture to your language and patterns.

---

## Auto-Prettier Hook

**Type**: Claude Code `PostToolUse` Hook (`~/.claude/hooks/`)

A simple but high-value hook: every time Claude edits or writes a file, Prettier auto-formats it. No more style drift, no more formatting commits.

**Configuration** (in `~/.claude/settings.json`):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

The hook reads the file path from Claude's tool input via `jq`, pipes it to `prettier --write`, and silently succeeds even if Prettier isn't configured for that file type. Zero friction.

**Requirements**: Node.js, Prettier installed in the project or globally.

---

## Installation

### Hooks

Copy hook files to your Claude Code hooks directory:

```bash
# RTK Rewrite
cp hooks/rtk-rewrite/rtk-rewrite.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/rtk-rewrite.sh
```

Then register in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/rtk-rewrite.sh"
          }
        ]
      }
    ]
  }
}
```

### Skills

Copy to your Claude Code skills directory:

```bash
cp -r skills/megaplan ~/.claude/skills/
```

### Commands

Copy to your Claude Code commands directory:

```bash
cp commands/handoff-create.md ~/.claude/commands/
cp commands/handoff-quick.md ~/.claude/commands/
cp commands/handoff-resume.md ~/.claude/commands/
```

### Lincoln Router

See [`lincoln-router/README.md`](lincoln-router/README.md) for architecture details. The implementation requires customization for your team's language and workflow patterns.

---

## Built With

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) -- Primary AI coding agent
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) -- Cross-provider review
- [Codex CLI](https://github.com/openai/codex) -- Cross-provider review
- [rtk](https://github.com/QuantGeekDev/rtk) -- Reduced Token Kit for CLI output compression

---

## About SallyLab

SallyLab is a 3-person team building AI-powered products. We believe the future of software development is human-AI collaboration, and we build tools to make that collaboration better.

- **Website**: [sallylab.io](https://sallylab.io)
- **Campaigns**: [campaigns.sallylab.io](https://campaigns.sallylab.io)

---

## License

MIT -- see [LICENSE](LICENSE) for details.

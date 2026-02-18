# Megaplan -- Tri-Provider Consensus Planning

A Claude Code skill that validates implementation plans across three independent AI providers: Claude, GPT (via Codex CLI), and Gemini (via Gemini CLI).

## Why

When you're making high-stakes architectural decisions, one AI's opinion isn't enough. Each provider has different training data, different strengths, and different blind spots. Megaplan leverages all three to produce plans that have been stress-tested from multiple angles.

## How It Works

1. **Claude Consensus**: Three Claude agents (Planner, Architect, Critic) iterate until they agree on a plan
2. **Cross-Provider Review**: The agreed plan is sent to Codex CLI and Gemini CLI in parallel for independent review
3. **Synthesis**: All feedback is compared -- agreements, disagreements, and suggestions are surfaced
4. **Resolution**: If there are major disagreements, the plan is refined and re-reviewed (max 2 iterations)
5. **Output**: Final plan with tri-provider confidence scores and a consolidated action item list

## Installation

```bash
cp -r megaplan ~/.claude/skills/
```

## Usage

In Claude Code, say:

```
megaplan: Design the authentication system for our mobile app
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Codex CLI](https://github.com/openai/codex) -- `npm install -g @openai/codex`
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) -- `npm install -g @google/gemini-cli`

See [SKILL.md](SKILL.md) for the full skill definition.

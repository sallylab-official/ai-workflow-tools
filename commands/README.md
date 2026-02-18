# Handoff Commands

Claude Code custom commands for AI-to-AI session continuity.

## The Problem

AI coding sessions lose all context when they end. The next session starts from zero -- no knowledge of what was tried, what failed, what decisions were made, or where to pick up. This wastes time and leads to repeated mistakes.

## The Solution

Three commands that create a structured handoff protocol:

| Command           | Purpose                                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------- |
| `/handoff-create` | Full handoff with goals, progress, failed approaches, decisions, and resume instructions |
| `/handoff-quick`  | Minimal 4-line handoff for simple tasks                                                  |
| `/handoff-resume` | Read a handoff, verify state, and continue from where the last session stopped           |

## Key Design Decision

**Failed approaches are mandatory.** Every handoff must document what was tried and abandoned, with specific reasons why. This is the single most valuable piece of information for the next session -- knowing what _not_ to do saves more time than knowing what to do.

## Installation

```bash
cp handoff-create.md ~/.claude/commands/
cp handoff-quick.md ~/.claude/commands/
cp handoff-resume.md ~/.claude/commands/
```

## Usage

In Claude Code:

```
/handoff-create          # End of session: create full handoff
/handoff-quick           # Quick context transfer
/handoff-resume          # Start of session: pick up from handoff
```

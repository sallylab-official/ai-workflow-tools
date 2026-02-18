# RTK Rewrite Hook

A Claude Code `PreToolUse` hook that transparently rewrites CLI commands to use [rtk](https://github.com/QuantGeekDev/rtk) (Reduced Token Kit), saving 60-90% of tokens on command output.

## How It Works

When Claude Code runs a Bash command, this hook intercepts the command before execution:

1. Reads the tool input JSON from stdin
2. Extracts the command string
3. Pattern-matches against known CLI tools
4. Rewrites the command to use the `rtk` prefix
5. Returns a JSON `updatedInput` that Claude Code applies transparently

Claude writes `git status` but `rtk git status` actually executes, returning compressed output.

## Supported Commands

| Category     | Commands                                                                                     |
| ------------ | -------------------------------------------------------------------------------------------- |
| Git          | `status`, `diff`, `log`, `add`, `commit`, `push`, `pull`, `branch`, `fetch`, `stash`, `show` |
| GitHub CLI   | `gh pr`, `gh issue`, `gh run`                                                                |
| JS/TS        | `vitest`, `tsc`, `eslint`, `prettier`, `playwright`, `prisma`                                |
| Python       | `pytest`, `ruff`, `pip`                                                                      |
| Go           | `go test`, `go build`, `go vet`, `golangci-lint`                                             |
| Rust         | `cargo test`, `cargo build`, `cargo clippy`                                                  |
| Containers   | `docker ps/images/logs`, `kubectl get/logs`                                                  |
| File ops     | `cat` (rewritten to `rtk read`), `grep`/`rg`, `ls`                                           |
| Network      | `curl`                                                                                       |
| Package mgmt | `pnpm list/ls/outdated`                                                                      |

## Safety

- Silently passes through if `rtk` or `jq` are not installed
- Skips commands that already use `rtk`
- Skips heredocs and complex command patterns
- Only rewrites the first command in a chain

## Installation

```bash
cp rtk-rewrite.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/rtk-rewrite.sh
```

Add to `~/.claude/settings.json`:

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

## Prerequisites

- [rtk](https://github.com/QuantGeekDev/rtk) -- `cargo install rtk`
- [jq](https://stedolan.github.io/jq/) -- `brew install jq` or `apt-get install jq`

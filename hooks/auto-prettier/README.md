# Auto-Prettier Hook

A Claude Code `PostToolUse` hook that auto-formats every file Claude edits or writes using Prettier.

## How It Works

After Claude uses the `Edit` or `Write` tool, this hook:

1. Reads the file path from the tool's JSON input via `jq`
2. Runs `npx prettier --write` on that file
3. Silently succeeds even if Prettier isn't configured for that file type

The result: every file Claude touches is automatically formatted. No style drift, no formatting commits.

## Configuration

Add to `~/.claude/settings.json`:

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

## Details

- **Trigger**: Runs after every `Edit` or `Write` tool use
- **Matcher**: `Edit|Write` -- regex pattern matching the tool name
- **Failure handling**: `|| true` ensures the hook never blocks Claude, even if Prettier fails or the file type isn't supported
- **No separate script needed**: The entire hook is a one-liner in settings.json

## Prerequisites

- Node.js
- Prettier (installed globally or in your project)

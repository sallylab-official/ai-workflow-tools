# Claude Code Completion Sound

Plays a sound when Claude Code finishes a task. Works via the `Stop` hook in Claude Code's settings.

## Install

### 1. Create sounds directory and copy the file

```bash
mkdir -p ~/.claude/sounds
cp reze-boom.wav ~/.claude/sounds/
```

### 2. Add the Stop hook to `~/.claude/settings.json`

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

If you already have other hooks configured, add the `Stop` block alongside them.

---

## Platform-specific play commands

Replace `afplay ...` in the hook with the command for your platform:

| Platform           | Command                                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| macOS              | `afplay --volume 1.0 ~/.claude/sounds/reze-boom.wav`                                                               |
| Linux (PulseAudio) | `paplay ~/.claude/sounds/reze-boom.wav`                                                                            |
| Linux (PipeWire)   | `pw-play ~/.claude/sounds/reze-boom.wav`                                                                           |
| Linux (ALSA)       | `aplay ~/.claude/sounds/reze-boom.wav`                                                                             |
| Windows (WSL)      | `powershell.exe -c "(New-Object Media.SoundPlayer '$env:USERPROFILE\\.claude\\sounds\\reze-boom.wav').PlaySync()"` |

**Full hook command for Linux (PulseAudio):**

```bash
[ -f ~/.claude/sounds/reze-boom.wav ] && paplay ~/.claude/sounds/reze-boom.wav 2>/dev/null &
```

The `[ -f ... ] &&` guard ensures the hook silently does nothing if the file doesn't exist. The trailing `&` runs playback in the background so Claude Code isn't blocked.

---

## Test it

Run this in your terminal to verify the sound plays:

```bash
# macOS
afplay ~/.claude/sounds/reze-boom.wav

# Linux
paplay ~/.claude/sounds/reze-boom.wav
```

---

## Use your own sound

You can replace `reze-boom.wav` with any `.wav` file. Just keep the filename or update the path in the hook command.

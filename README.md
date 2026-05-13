# claude-shush

Silences the **"task tools haven't been used recently..."** system reminder
that Claude Code injects when it thinks you should be tracking your work.

The reminder is hardcoded into the compiled `claude` binary. There's no
documented setting, env var, or hook to suppress it. This script blanks
the reminder template in place, then re-signs the binary ad-hoc so macOS
will still launch it.

The reminder still "fires" mechanically inside the harness, but its content
is whitespace — so the model doesn't see a nudge, and you don't drift toward
capitulating to it.

## Usage

```sh
./claude-shush          # patch the resolved `claude` binary
./claude-shush -n       # dry run: report findings, modify nothing
./claude-shush /path    # patch a specific binary
```

Backups land next to the original as `<binary>.bak.<YYYYMMDDTHHMMSS>`.

## What it does

1. Resolves the `claude` binary (or takes a path argument).
2. Finds the live reminder template inside the binary by anchor strings.
3. Overwrites the template content with spaces of equal length, preserving
   binary layout (so offsets and codesigning hashes are the only things
   that change).
4. On macOS, re-signs ad-hoc (`codesign --force --sign -`).

## macOS caveats

Anthropic ships `claude` with a Developer ID signature and hardened runtime.
Patching invalidates the original signature, so the script re-signs ad-hoc.

- **Keychain access:** on first launch after patching, macOS prompts for
  access to claude's stored credentials. Click **Always Allow** once per
  binary version. (The ad-hoc signature's identity is the binary hash, so
  the trust persists for that exact patched file.)
- **Updates:** Claude Code installs each release as a separate file (e.g.
  `~/.local/share/claude/versions/2.1.140/`). After an update, rerun this
  script on the new binary. You'll get one more Keychain prompt.

## Linux

No codesigning step needed. The patch alone is sufficient.

## Reverting

```sh
cp /path/to/claude.bak.<timestamp> /path/to/claude
```

On macOS, the backup retains the original Anthropic signature, so no
re-sign is needed after restore.

## Why

The reminder fires on a coarse heuristic ("no task tools used recently")
without regard for whether the current work actually needs tracking. On
short tasks it lands as a non-sequitur, and repeated firings can erode
judgment — eventually you (or the model) capitulate to silence the prompt
instead of trusting "this is two steps, no checklist needed."

If you'd rather request a real setting from Anthropic, open an issue at
<https://github.com/anthropics/claude-code/issues> proposing something
like `suppressTaskReminder: true` in `settings.json`.

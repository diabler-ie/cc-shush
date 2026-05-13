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

## Suggested CLAUDE.md addition

After installing, add a note to `~/.claude/CLAUDE.md` so future sessions
know the binary is patched (and don't get confused when a reminder they
might otherwise expect doesn't fire):

```md
## Local environment

The `claude` binary on this machine is patched via `claude-shush` to blank
the "task tools haven't been used recently..." system reminder. Re-run
`claude-shush` after each Claude Code update.

Task tools are worth reaching for on 4+ step work or when you're juggling
independent threads. Skip them for 1-3 step operations.
```

The second paragraph is a replacement heuristic — it gives Claude a
concrete threshold to anchor to instead of just removing the nudge and
leaving nothing in its place. Tune the numbers (4+ steps, 1-3 step cap)
to match your own workflow.

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

- **Keychain access:** in practice, the patched binary tends to launch
  without prompting — Claude Code's Keychain item is created with a
  permissive ACL that allows ad-hoc-signed binaries to read stored
  credentials. If your Keychain is configured more strictly, macOS will
  prompt on first launch; click **Always Allow** to remember the new
  signature for that exact binary.
- **Updates:** Claude Code installs each release as a separate file (e.g.
  `~/.local/share/claude/versions/2.1.140/`). After an update, rerun this
  script on the new binary.

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

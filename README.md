# cc-shush

Patches Claude Code's compiled binary to fix two system reminders that
don't carry their weight:

1. **Silences the "task tools haven't been used recently..."** nudge —
   the reminder still fires mechanically, but its content is whitespace,
   so the model doesn't see a judgment-eroding signal on short tasks.
2. **Softens the "IMPORTANT: ... you MUST ... Do not ignore it." stack**
   on the reminder that fires when you type a message while a tool is
   running. Same information, neutral tone.

Neither reminder is suppressible via documented settings, env vars, or
hooks. This script byte-edits the binary in place and re-signs ad-hoc
on macOS so it still launches.

## Usage

```sh
./cc-shush          # patch the resolved `claude` binary
./cc-shush -n       # dry run: report findings, modify nothing
./cc-shush /path    # patch a specific binary
```

Backups land next to the original as `<binary>.bak.<YYYYMMDDTHHMMSS>`.

## Automating with `cc-shush-watch` (macOS)

Claude Code installs new versions roughly daily as separate files under
`~/.local/share/claude/versions/`. `cc-shush-watch` automates re-running
the patch via launchd:

```sh
./cc-shush-watch --install     # register the launchd agent
./cc-shush-watch --uninstall   # remove it
./cc-shush-watch --help        # show options
```

What `--install` does:

- Writes `~/Library/LaunchAgents/com.cc-shush.watch.plist` and loads it.
- The agent watches `~/.local/share/claude/versions/` and fires whenever
  the directory changes (i.e., a new version is installed).
- On each fire, the wrapper resolves the current `claude` binary and
  checks its codesign signature:
  - **Ad-hoc** → already patched by an earlier run; exit silently.
  - **Anthropic Developer ID** → run `cc-shush` and notify on success.
  - **Anything else** → log and skip (won't touch unknown binaries).
- If `cc-shush` finds zero patchable anchors on an Anthropic-signed
  binary, **notifies you that the patches are broken** — Anthropic has
  changed the reminder wording and the script needs updating.

Notifications use macOS `display notification` and appear in Notification
Center. Logs accumulate at `~/.local/state/cc-shush/watch.log`.

## Suggested CLAUDE.md addition

After installing, add a note to `~/.claude/CLAUDE.md` so future sessions
know the binary is patched (and don't get confused when a reminder they
might otherwise expect doesn't fire):

```md
## Local environment

The `claude` binary on this machine is patched via `cc-shush` to blank
the "task tools haven't been used recently..." system reminder. Re-run
`cc-shush` after each Claude Code update.

Task tools are worth reaching for on 4+ step work or when you're juggling
independent threads. Skip them for 1-3 step operations.
```

The second paragraph is a replacement heuristic — it gives Claude a
concrete threshold to anchor to instead of just removing the nudge and
leaving nothing in its place. Tune the numbers (4+ steps, 1-3 step cap)
to match your own workflow.

## What it does

1. Resolves the `claude` binary (or takes a path argument).
2. Walks a list of patches, each defined as either an **anchor-span**
   (start + end anchor strings, used when the template contains runtime
   `${...}` interpolations to preserve) or a **literal-replace** (exact
   string match, used for fixed text like the IMPORTANT-MUST line).
3. For anchor-spans, overwrites the matched span with spaces. For
   literal-replaces, substitutes the new text and pads to the original
   length. Either way, total binary length is preserved.
4. On macOS, re-signs ad-hoc (`codesign --force --sign -`).

Patches are declared in a list at the top of the script — add more
entries if you find other reminders worth retuning.

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

**Task-tool reminder:** fires on a coarse heuristic ("no task tools used
recently") without regard for whether the current work actually needs
tracking. On short tasks it lands as a non-sequitur, and repeated firings
erode judgment — eventually you (or the model) capitulate to silence the
prompt instead of trusting "this is two steps, no checklist needed."

**User-typed-mid-tool reminder:** the information is fine — "user added
input mid-tool, address it after the current operation." But the original
wraps it in IMPORTANT + you MUST + Do not ignore it, which reads like the
harness defaults to assuming the model would otherwise plow through and
ignore the user. Same signal, neutral framing.

If you'd rather request real settings from Anthropic, open an issue at
<https://github.com/anthropics/claude-code/issues>.

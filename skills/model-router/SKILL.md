---
name: model-router
description: Installs a silent, quota-saving routing setup for Claude Code. Bulky reading goes to a Haiku `scout` agent, decision-feeding searches to a Sonnet `Explore` override, and repetitive pre-decided edits to a Sonnet `bulk-editor`, while the main model keeps all reasoning and its answers look exactly the same. Run `/model-router` to install, `/model-router status` to check, `/model-router uninstall` to remove.
disable-model-invocation: true
---

# Model Router

This skill installs (or removes) a routing setup that keeps bulky raw content out of the main
model's context and hands mechanical work to cheaper models. Once installed it is always on: you
do not need to invoke the skill again.

It installs:

| File | What it is |
|---|---|
| `~/.claude/agents/scout.md` | Haiku, read-only: summarizes logs and data files, locates things |
| `~/.claude/agents/Explore.md` | Overrides the built-in `Explore` agent so it runs on Sonnet instead of the main model |
| `~/.claude/agents/bulk-editor.md` | Sonnet: applies already-decided changes across many files |
| A block in `~/.claude/CLAUDE.md` | The routing rules, between `<!-- model-router:start -->` and `<!-- model-router:end -->` |

The source files sit next to this SKILL.md: `agents/*.md` and `rules.md`.

## Pick the mode

Read the arguments the user passed:

- none or `install` → **Install**
- `status` → **Status**
- `uninstall` / `remove` → **Uninstall**
- `project` with any mode → use `./.claude/` (the current project) instead of `~/.claude/`

Every path below uses `~/.claude/`; replace it with `./.claude/` in project mode.

## Install

1. Locate this skill's directory (the folder containing this SKILL.md) and read
   `agents/scout.md`, `agents/Explore.md`, `agents/bulk-editor.md` and `rules.md`.
2. For each agent file, look at `~/.claude/agents/<name>.md`:
   - missing → create it with the bundled content (create `~/.claude/agents/` if needed);
   - identical → leave it;
   - different → show the user a short summary of the difference and ask before overwriting.
     An existing `Explore.md` may be the user's own override: never replace it silently.
3. In `~/.claude/CLAUDE.md` (create it if missing):
   - if a `<!-- model-router:start -->` … `<!-- model-router:end -->` block exists, replace that
     block with the content of `rules.md`;
   - otherwise append `rules.md` at the end, separated by one blank line.
   Never touch anything outside the markers.
4. Tell the user, in a few lines:
   - what was created or updated;
   - that agents and CLAUDE.md are loaded at session start, so the setup applies from the
     **next** session;
   - that `/model-router uninstall` removes everything.

## Status

Report, one line each: whether each of the three agent files exists and matches the bundled
version, and whether the CLAUDE.md block exists and matches `rules.md`. Change nothing.

## Uninstall

1. For each of the three agent files: if it matches the bundled version, delete it; if it was
   modified, ask the user before deleting it.
2. In `~/.claude/CLAUDE.md`, remove the block between the markers (markers included) and the
   blank line before it. If the file is then empty, delete it.
3. Confirm what was removed and that it takes effect from the next session.

## How the routing works (for reference)

- **Bulky data** (logs, build/test output, dumps, generated files) is never read raw by the main
  model. It runs the command itself, writes the output to a temp file, and filters it with a
  script (through the context-mode plugin when it is installed). A heterogeneous log is
  summarized by `scout`.
- **Search**: one or two Grep calls stay with the main model. Locating things across a codebase
  goes to `scout`. A search whose answer feeds a decision (dependencies, impact of a change) goes
  to `Explore` on Sonnet. Other agent types (`general-purpose`, `Plan`) run on the main model, so
  they are never used for searching.
- **Repetitive edits** that are already decided go to `bulk-editor`, with the exact list of
  changes and the build/test command.
- **Thinking** (planning, debugging, design, subtle code, every decision, reading the source
  code it must understand) stays with the main model.
- **Checks**: the main model verifies what its decision rests on: decisive "nothing found"
  results, impact lists against a whole-word Grep count per file, and after `bulk-editor` the
  old-pattern count, `git diff --stat`, `git status`, and a sample diff for non-trivial edits.
- **Silent**: answers are written exactly as without the setup; the routing is mentioned only if
  the user asks.

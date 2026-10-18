# model-router

**Spend less of your Claude Code session limit on Opus, without changing what you see.**

When you pick Opus, everything runs on Opus: reading a 20,000-line build log, grepping a
codebase, renaming a function in 40 files. `model-router` installs a small routing setup so
that bulky, mechanical work goes to cheaper models, while Opus keeps all the reasoning. Your
answers look exactly the same. The only difference shows up in your usage.

## Install

```bash
npx skills add imskyleen/model-router -g -a claude-code
```

Then, in Claude Code:

```
/model-router
```

Start a new session: the setup is loaded at session start and is always on from then on.
You never need to invoke the skill again.

| Command | Effect |
|---|---|
| `/model-router` | Install (asks before overwriting any file you changed) |
| `/model-router status` | Check what is installed |
| `/model-router uninstall` | Remove everything it added |
| `/model-router project` | Install into the current project's `.claude/` instead of `~/.claude/` |

## How it works

The main model stays in charge but follows one rule: **don't read what is long and needs no
thinking. Hand it off.**

| Task | Who does it |
|---|---|
| Filter a log, count errors | A script (via [context-mode](https://github.com/mksglu/context-mode) if installed). No model reads the raw text |
| Summarize a big log, locate things across a codebase | **`scout`**, on Haiku |
| A search that feeds a decision ("what breaks if I change this?") | **`Explore`**, on Sonnet |
| Apply an already-decided change to many files | **`bulk-editor`**, on Sonnet |
| Planning, debugging, design, subtle code, every decision | **The main model** (Opus) |

Helpers return short, size-capped summaries, and the main model checks what its decision rests
on before answering: it re-checks "nothing found" results, compares impact lists with a Grep
count file by file, and reviews the diff after bulk edits.

## Why it saves quota

1. **The main context stays small.** Everything Opus reads stays in its context until the end of
   the session and adds weight to every later turn. When `scout` reads a 20,000-token log and
   returns 10 lines, only those 10 lines reach Opus.
2. **`Explore` stops running on Opus.** Claude Code's built-in `Explore` agent, which Claude
   launches on its own for searches, runs on your main model. This setup overrides it to run on
   Sonnet.
3. **Quality is kept.** Decisions and hard code stay with Opus, and helpers are only used where
   a mistake would be easy to spot.
4. **It's invisible.** Same answer style, no mention of the routing unless you ask, and no temp
   files in your repo.

## What it installs

| File | Content |
|---|---|
| `~/.claude/agents/scout.md` | Haiku, read-only. Cannot run builds, tests or installs |
| `~/.claude/agents/Explore.md` | Overrides the built-in `Explore` with a Sonnet version |
| `~/.claude/agents/bulk-editor.md` | Sonnet. Never runs `git checkout/reset/stash/clean` or `rm`, never commits, stops instead of "fixing" a failing build |
| A block in `~/.claude/CLAUDE.md` | The routing rules, between `<!-- model-router:start -->` markers. Nothing else in the file is touched |

The source of each file is in [`skills/model-router/`](skills/model-router/).

## Limits

- **Claude Code only.** It relies on Claude Code subagents and `CLAUDE.md`.
- **The main loop still runs on Opus.** Every message you send goes through it. Savings come from
  what it no longer has to read, so sessions heavy on logs, searches and repetitive edits gain
  the most. A pure design discussion gains little.
- **Subagent calls are visible** as tool cards in the transcript. Answer text is not affected.
- **`Explore` is overridden globally.** If you already have your own `Explore.md`, the installer
  asks before replacing it.

## Measure it

Run the same heavy task (for example "find out why the build fails") in a session without the
setup and in one with it, then compare your usage.

## License

[MIT](LICENSE)

# code-humanizer

**[humanizer](https://github.com/blader/humanizer), but for code.** An agent skill that removes signs of AI-generated code from a repository — the structural slop coding agents leave behind when they optimize for "tests pass" instead of "codebase stays healthy."

The prose humanizer catalogs AI *writing* tells (em-dashes, "it's not just X, it's Y", rule-of-three). This catalogs AI *coding* tells:

- **Reimplementing a helper the repo already has** (the signature tell — agents write from the prompt outward, not from the repo inward)
- `_v2` / `_new` / `_enhanced` clones of functions the agent didn't dare modify
- `try: import ujson except ImportError: import json` fallbacks nobody asked for
- `except Exception: return ""` — bugs converted into silently-wrong output
- Abstract base classes and registries with exactly one implementation
- Dead "for future use" helpers, wrappers that add nothing, global flags read from one place
- `hasattr`/`getattr` probing ladders, paranoid re-validation
- Comments that narrate the next line, docstrings that restate the function name

16 numbered patterns in 5 tiers, each with detection signals and before/after examples — plus the part that matters more than the catalog:

## Why it's not just a pattern list

Modern agents already *recognize* most slop when pointed at a file. Where they fail is discipline. In our baseline test, an agent without this skill cleaned a slop file nicely — and **silently changed a public error type along the way** (swapped an `AttributeError` for a "nicer" `ValueError`), in one un-reviewable mega-change, editing before ever running the tests.

So the skill's core is three iron rules the catalog hangs off:

1. **Behavior preservation is absolute** — including error types and timing. Latent bugs get *reported*, never silently "improved."
2. **No tests → no edits.** The test suite is the oracle for "meaning-preserving." Missing oracle = report-only mode.
3. **One pattern-class per commit**, suite run after each, behavior-risk changes isolated in `[BEHAVIOR]`-labeled commits.

Plus a **false-positive guard**: severity 1 = "present but justified" (fallbacks with documented reasons, defensive code at trust boundaries, plugin registries, migration-period `_v2`s) — those are exempt. The goal is a healthier repo, not a body count.

## Install

Copy (or clone) this directory into your agent's skills folder:

```bash
# Claude Code
git clone https://github.com/LeonardNJU/code-humanizer ~/.claude/skills/code-humanizer
```

Any harness that reads `SKILL.md` agent skills works the same way — the skill is a single Markdown file, no build step, no dependencies.

## Use

```
> use code-humanizer to scan this PR
> deslop pkg/report.py — you have my approval to fix
> this repo was vibe-coded, humanize it (report first)
```

Default mode is **scan → report** (findings table with pattern #, severity 0–4, evidence, proposed fix, behavior risk). Fix mode runs on your approval, one pattern per commit, tests green after every step.

## Scope honesty

- Examples are Python; the patterns and workflow are language-agnostic (signals sections mention Python idioms — port as needed).
- This removes *structural* debt, not formatting opinions — that's your linter's job.
- Judgment-heavy debt (root-cause-vs-workaround fixes) is reported, not auto-fixed.

## Credits

Pattern-catalog format inspired by [blader/humanizer](https://github.com/blader/humanizer). The debt taxonomy distills a research project on agent-induced technical debt (correctness-equivalent patch analysis); the iron rules come from watching capable agents fail without them.

MIT.

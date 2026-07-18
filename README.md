# code-humanizer
<img width="1280" height="640" alt="social-preview" src="https://github.com/user-attachments/assets/ae4cb1c9-dbc7-4828-a814-725f08a20001" />


English | [中文](README.zh.md)

[![Stars](https://img.shields.io/github/stars/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/stargazers) [![License](https://img.shields.io/github/license/LeonardNJU/code-humanizer?style=flat-square)](LICENSE) [![Last commit](https://img.shields.io/github/last-commit/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/commits) [![Issues](https://img.shields.io/github/issues/LeonardNJU/code-humanizer?style=flat-square)](https://github.com/LeonardNJU/code-humanizer/issues)

**[humanizer](https://github.com/blader/humanizer), but for code.** An agent skill that removes signs of AI-generated code from a repository — the structural slop coding agents leave behind when they optimize for "tests pass" instead of "codebase stays healthy."

## Install

One command, picks your agents for you (Claude Code, Codex, Cursor, Cline, Gemini, Copilot, and more):

```bash
npx skills add LeonardNJU/code-humanizer
```

Or drop it in by hand — the skill is a single `SKILL.md`, no build step, no dependencies:

```bash
# Claude Code
git clone https://github.com/LeonardNJU/code-humanizer ~/.claude/skills/code-humanizer
```

Any harness that reads `SKILL.md` agent skills works the same way.

## Use

```
> use code-humanizer to scan this PR
> deslop pkg/report.py — you have my approval to fix
> this repo was vibe-coded, humanize it (report first)
```

Default mode is **scan → report** (findings table with pattern #, severity 0–4, evidence, proposed fix, behavior risk). Fix mode runs on your approval, one pattern per commit, tests green after every step.

## What it catches

The prose humanizer catalogs AI *writing* tells (em-dashes, "it's not just X, it's Y", rule-of-three). This catalogs AI *coding* tells — 16 numbered patterns in 5 tiers, each with detection signals and before/after examples in [SKILL.md](SKILL.md):

### The patterns

**Tier 1 — Duplication and reinvention**

| # | Pattern | The tell |
|---|---------|----------|
| 1 | **Reimplementing an existing helper** *(the signature tell)* | a new private function that duplicates something in `utils`/a sibling module — agents write from the prompt outward, not from the repo inward |
| 2 | **`_v2` / `_new` / `_impl` clones** | `foo` and `foo_v2` both alive; the agent didn't dare modify the original, so now there are two sources of truth |
| 3 | **Reinventing stdlib / installed deps** | hand-rolled `groupby`, deep-copy-via-JSON, manual URL parsing |

**Tier 2 — Speculative architecture**

| # | Pattern | The tell |
|---|---------|----------|
| 4 | **Single-implementation abstraction** | an ABC / registry / "pluggable backend" with exactly one implementation, one registration, one call site |
| 5 | **Dead "for future use" code** | helpers with no call site; docstrings saying *flexible, extensible, seamlessly* |
| 6 | **Wrapper that adds nothing** | a function whose body is one same-argument call |
| 7 | **Config/API sprawl for a local case** | a new global flag or public parameter consulted from exactly one place |

**Tier 3 — Defensive slop**

| # | Pattern | The tell |
|---|---------|----------|
| 8 | **Broad exception swallowing** | `except Exception: return ""` — crashes (visible, debuggable) converted into corruption (invisible) |
| 9 | **Unjustified try-import fallback** | `try: import ujson except ImportError: import json` with no benchmark, no extras entry, no fallback test |
| 10 | **Attribute-probing chains** | `hasattr`/`getattr`/`isinstance` ladders accepting "dict or object or maybe None" |
| 11 | **Paranoid re-validation** | `if x is not None` on values that were just constructed |

**Tier 4 — Noise**

| # | Pattern | The tell |
|---|---------|----------|
| 12 | **Narrating comments** | the comment restates the next line (`# Join the rows with newlines`) |
| 13 | **Boilerplate docstrings** | the docstring is the function name with spaces; *robust, comprehensive, seamless* |
| 14 | **Dead imports, unused variables, banners** | leftovers from deleted attempts; `# ===== SECTION =====`; stray debug prints |

**Tier 5 — Test slop** *(report-only by default)*

| # | Pattern | The tell |
|---|---------|----------|
| 15 | **Tests that assert the mock** | every collaborator mocked; the test can never fail for a real reason |
| 16 | **Trivial or duplicated assertions** | asserting literals; the same case re-tested under three names |

Every finding gets a severity 0–4, where **1 = present but justified → exempt**: fallbacks with documented reasons, defensive code at trust boundaries, plugin registries, migration-period `_v2`s stay untouched. The catalog is half the skill — the other half matters more:

## Why it's not just a pattern list

Modern agents already *recognize* most slop when pointed at a file. Where they fail is discipline. In our baseline test, an agent without this skill cleaned a slop file nicely — and **silently changed a public error type along the way** (swapped an `AttributeError` for a "nicer" `ValueError`), in one un-reviewable mega-change, editing before ever running the tests.

So the skill's core is three iron rules the catalog hangs off:

1. **Behavior preservation is absolute** — including error types and timing. Latent bugs get *reported*, never silently "improved."
2. **No tests → no edits.** The test suite is the oracle for "meaning-preserving." Missing oracle = report-only mode.
3. **One pattern-class per commit**, suite run after each, behavior-risk changes isolated in `[BEHAVIOR]`-labeled commits.

Plus a **false-positive guard**: severity 1 = "present but justified" (fallbacks with documented reasons, defensive code at trust boundaries, plugin registries, migration-period `_v2`s) — those are exempt. The goal is a healthier repo, not a body count.

## Real-world run

First field test: a private ML research repo, 13.4k LOC of Python (21-module package + 36 experiment scripts + 20 test files), written largely by coding agents under human review. Scan mode, zero edits, `git status` clean before and after.

**24 findings — and a sharp profile.** Defensive slop (broad excepts, try-import fallbacks, probing chains, narrating comments): **zero**. Test slop: **zero across all 20 test files**. The debt was almost entirely **Tier-1 duplication** (11 findings, 40+ pasted instances), and the copies were already biting:

- a provenance helper pasted into **20 of 22 run scripts had already drifted** — 3 copies gained an env-var fallback the other 17 lack;
- an experiment class copy-pasted into a "learned" variant whose metric method **silently diverged** — one copy checks 3 intervention pairs, the other 1;
- the same orthogonal-matrix sampler pasted 4×, the same query dataclass pasted verbatim across modules.

4 findings were exempted as justified (severity 1): a preregistration guard, a deliberate sampling-strategy comparison, bounded retries with a final `raise`. 8 of 36 scripts and 6 of 21 modules came back fully clean — and were reported as clean.

The take-away matched the skill's premise: agents working under review don't swallow errors — they **rewrite what already exists**. Repo-context duplication detection is the tier that matters.

## Scope honesty

- Examples are Python; the patterns and workflow are language-agnostic (signals sections mention Python idioms — port as needed).
- This removes *structural* debt, not formatting opinions — that's your linter's job.
- Judgment-heavy debt (root-cause-vs-workaround fixes) is reported, not auto-fixed.

## Contributing patterns

The catalog is 16 patterns today and will never be finished — agents keep inventing new kinds of slop. If you've hit one that isn't covered, [open an issue](https://github.com/LeonardNJU/code-humanizer/issues) with a minimal before/after example (or a PR — even better). New patterns enter the catalog the same way the original 16 did: a real failing example first, then the rule.

## Star History

<a href="https://www.star-history.com/?repos=LeonardNJU%2Fcode-humanizer&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&theme=dark&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=LeonardNJU/code-humanizer&type=date&legend=top-left&sealed_token=PxMzEEMfOA3R97IHVMG0zQqC1xVjuoA3cMbkfPzjtmhwAbL9n3oQVqHhhzNgKOwqwD0CShQpk_ostzzb2_m-8qA_U5IFphEK5su_nHoI1HKzcVq3-8oq8HZGL79TUmv5WtGBpBCqOtdrgrF8_KKxta_MCbS9IscnCEtMju92w8_qG8TtIZw_zZ68xYli" />
 </picture>
</a>

## Credits

Pattern-catalog format inspired by [blader/humanizer](https://github.com/blader/humanizer). The debt taxonomy distills a research project on agent-induced technical debt (correctness-equivalent patch analysis); the iron rules come from watching capable agents fail without them.

## License

[MIT](LICENSE) — use it, fork it, vendor it into your team's skills directory, rewrite the catalog for your stack, ship it inside something commercial. The only obligation is keeping the license notice. If it saved your repo from a `_v2`, a star is appreciated — never required.

## Community

All feedback and new-pattern reports go through [GitHub issues](https://github.com/LeonardNJU/code-humanizer/issues).

Introduction threads: [linux.do](https://linux.do/t/topic/2590134/) · [NJU-AIA forum](https://forum.nju-aia.com/t/topic/265)

友链:[linux.do](https://linux.do)

---
name: code-humanizer
version: 0.2.0
description: Use when AI-written code needs structural cleanup or when implementing a change under a guard against AI coding slop — duplicated reimplementations of existing helpers, try-import fallbacks, broad exception swallowing, speculative abstraction layers, _v2 copies, dead "for future use" code, narrating comments — including requests to "deslop", "remove AI slop", "humanize code", review an AI-generated PR beyond tests passing, or prevent these patterns in the current change.
license: MIT
compatibility: any-agent
---

# Code Humanizer

Remove signs of AI-generated code from a repository, or prevent the same patterns in the current change with Guard mode. The prose [humanizer](https://github.com/blader/humanizer) removes AI *writing* patterns; this handles AI *coding* patterns — the structural debt agents leave behind when they optimize for "tests pass" instead of "codebase stays healthy."

**Core principle:** AI code slop is not ugly code — it is code that *works* but degrades the repository: it reimplements what already exists, adds abstraction nobody asked for, swallows errors it should surface, and hedges against situations that cannot occur. Unlike prose, code has an oracle: **the test suite decides what "meaning-preserving" means. Use it constantly.**

## Iron rules

Rules 1–4 govern **cleanup of pre-existing code** in Modes A/B. In Guard mode they apply to any cleanup performed after implementation, not to the requested feature work itself. Rule 5 applies in every mode.

1. **Behavior preservation is absolute.** "Cleanup" that changes behavior is a bug with good intentions. This includes *error types and error timing* — swapping an accidental `AttributeError` for a "nicer" `ValueError` changes behavior for every caller that catches it. If you find a latent bug or an ugly-but-load-bearing behavior: **flag it in the report; never fix it silently as part of cleanup.**
2. **No tests → no cleanup edits.** Run the test suite first. If it doesn't exist, doesn't pass, or doesn't cover the pre-existing code you'd clean, you may only **report** that debt (Mode A). Offer to write characterization tests first. This does not forbid implementing a requested change in a project whose tests are incomplete; report the verification gap.
3. **Report before rewriting pre-existing code.** Default to Mode A (scan → report). Only enter Mode B (fix) when the maintainer approves, or was explicit that they want fixes. Guard mode may directly avoid or remove slop introduced by the current change.
4. **One pattern-class per cleanup commit.** Each commit that removes pre-existing slop handles one kind and passes the full suite. A red test reverts the commit — do not "fix forward" into unrelated code. Ordinary feature commits under Guard mode need not be split by pattern when the prevention is inseparable from the implementation.
5. **Search before you judge duplication.** Build a mental index of the repo's existing helpers *before* scanning or implementing (grep `utils`, `helpers`, `common`, validators, existing base classes). You cannot recognize a reimplementation if you don't know what exists.

## Pattern catalog

Severity per finding: **0** absent · **1** present but justified (→ exempt, do not touch) · **2** minor debt · **3** clear debt, raises future maintenance cost · **4** severe, breaks semantics or architecture boundaries.

### Tier 1 — Duplication and reinvention

**1. Reimplementing an existing helper** *(the signature AI tell)*
**Signals:** new private function whose body resembles something in `utils`/`helpers`/a sibling module; a regex/validation/parsing/formatting block that the repo already owns; new code that imports nothing from the package it lives in.
**Problem:** agents write code from the prompt outward, not from the repo inward. Every duplicate forks future fixes.
```python
# Before — repo already has utils.validate_email
def _is_valid_email(s):
    pattern = r"^[\w.+-]+@[\w-]+\.[\w.]+$"
    if re.match(pattern, s):
        return True
    return False

# After
from .utils import validate_email
```

**2. `_v2` / `_new` / `_impl` clones**
**Signals:** `foo` and `foo_v2` both alive; `enhanced_`, `improved_`, `new_` prefixes; copy-pasted body with one branch changed.
**Problem:** the agent didn't dare modify the original, so now there are two sources of truth. Merge them (a parameter or the newer body), keep one name, unless a documented migration is in progress.

**3. Reinventing the standard library / installed deps**
**Signals:** hand-rolled `groupby`, deep-copy-via-JSON, manual URL parsing, bespoke date math; the equivalent is in `stdlib` or an already-installed dependency.

### Tier 2 — Speculative architecture

**4. Single-implementation abstraction**
**Signals:** ABC / interface / `Protocol` with exactly one concrete class; a registry or factory with exactly one registration; "pluggable backend" used from one call site.
**Problem:** flexibility for a future that was never requested. Inline it; the abstraction can return *when the second implementation arrives*.
```python
# Before: AbstractReportBackend + BackendRegistry + TextReportBackend (3 classes, 40 lines)
# After:
_BACKENDS = {"text": lambda rows: "\n".join(map(str, rows))}
```

**5. Dead "for future use" code**
**Signals:** helpers with no call site; parameters no caller passes; `export_*`/`*_to_dict` "provided for extensibility"; docstrings saying *flexible, extensible, seamlessly*.
**Problem:** unreachable code still costs review, grep hits, and false confidence. Delete it — git remembers. **Grep the whole repo (and scripts, and templates) for usages first.**

**6. Wrapper that adds nothing**
**Signals:** function whose body is a single call with the same arguments; class inheriting only to `super()` everything.

**7. Config/API sprawl for a local case**
**Signals:** new global flag, config field, CLI arg, or public parameter consulted from exactly one place; module-level constant that only one function reads.
**Problem:** every global knob multiplies the state space forever. Make it local, or a parameter of the one function that cares.

### Tier 3 — Defensive slop

**8. Broad exception swallowing**
**Signals:** `except Exception:`/bare `except` that returns a default, logs nothing, or `pass`es; `.get()` chains that convert bugs into silently-wrong output.
**Problem:** converts crashes (visible, debuggable) into corruption (invisible). Narrow to the exceptions that can actually occur, or let it raise. **Careful — this is behavior; may need maintainer sign-off (Iron rule 1).**

**9. Unjustified try-import fallback**
**Signals:** `try: import fast_x except ImportError: import x` with no benchmark, no extras entry in packaging metadata, no test covering the fallback path; optional-dependency dances for deps that are not optional.
**Problem:** two code paths, one tested. If the dependency matters, declare it; if not, drop it.

**10. Attribute-probing chains**
**Signals:** `hasattr`/`getattr`/`isinstance` ladders to accept "dict or object or maybe None"; the same probing expression copy-pasted at every field access.
**Problem:** the agent didn't check what type actually flows here, so it accepted everything. Find the real type (tests tell you), then write to it.

**11. Paranoid re-validation**
**Signals:** `if x is not None` on values that were just constructed; re-checking invariants the type system or an upstream gate already guarantees.

### Tier 4 — Noise

**12. Narrating comments**
**Signals:** comment restates the next line (`# Join the rows with newlines`); comments addressed to the reviewer (`# This change ensures correctness`); TODO the agent wrote and resolved in the same PR.
**Rule:** a comment earns its line only by stating something the code *cannot* say (constraint, invariant, why-not-the-obvious-way).

**13. Boilerplate docstrings**
**Signals:** docstring is the function name with spaces (`def get_user(): """Get the user."""`); marketing adjectives (*robust, comprehensive, seamless*).

**14. Dead imports, unused variables, decorative banners**
**Signals:** imports left from deleted attempts; `# ===== SECTION =====` banners; emoji in identifiers/log lines; leftover `print`/debug logging.

### Tier 5 — Test slop (report-only by default)

**15. Tests that assert the mock** — every collaborator mocked, the assertion checks the mock was called; the test can never fail for a real reason.
**16. Trivial or duplicated assertions** — asserting literals, or the same case re-tested under three names to inflate coverage.

## What NOT to flag (false-positive guards)

Severity 1 = justified. The pattern's *shape* is not the crime; the *lack of justification* is. Look for justification before flagging:

- **Fallbacks/compat shims with a reason** — documented platform differences, packaged extras (`pip install pkg[fast]`), version gates during a migration. The tell is *unjustified*, not *fallback*.
- **Defensive code at trust boundaries** — parsing user input, network payloads, plugin-supplied objects. Probing and broad-catch are legitimate exactly where data is untrusted (but should still log, not `pass`).
- **Registries/ABCs in actual plugin systems** — if entry points load implementations dynamically, one *in-repo* implementation doesn't mean single-implementation.
- **Duplication with a measured reason** — a hot-path copy with a benchmark comment; vendored code kept intentionally in sync.
- **`_v2` during a documented migration** — check git history / CHANGELOG before merging.
- **Error swallowing that logs and is commented** — a deliberate resilience decision is the maintainer's to revisit, not yours.
- **Style you merely dislike.** This skill removes *structural debt*, not formatting opinions — that's the linter's job.

When in doubt: report at severity 1–2, don't fix. **Look for clusters** — one narrating comment is nothing; narrating comments + a single-impl registry + a `_v2` + an unused export in the same PR is a confession.

## Workflow

**Mode A — Scan (default):**
1. **Oracle check:** run the test suite; record pass state and rough coverage of target files.
2. **Index:** map the repo's existing helpers/abstractions (this powers pattern 1).
3. **Scan:** walk the target (a diff, a PR, a module, or the whole repo) against the catalog. For each finding: `pattern # · file:line · severity · evidence (one line) · proposed fix · behavior risk (none / error-type / needs-maintainer)`.
4. **Report:** findings grouped by pattern, severities summed per file, exemptions listed with their justification. End with the proposed fix order (highest severity, lowest behavior-risk first).

**Mode B — Fix (on approval):**
5. For each pattern-class, in the approved order: fix all instances → run the full suite → commit (`deslop: <pattern> (#N) — <n> instances`). Red tests revert the commit.
6. Anything with behavior risk (error types, swallowed exceptions, unknown-key paths) is **proposed as its own clearly-labeled commit or left as a report item** — never mixed into safe cleanups.
7. **Audit pass:** re-scan the result and ask *"what would still make a reviewer say an AI wrote this?"* Fix or report the remainder.
8. **Summary:** per-pattern counts removed, LOC delta, exemptions honored, behavior-risk items awaiting the maintainer.

**Mode C — Guard (while implementing):**

Enter Guard mode when the user asks to prevent AI coding slop while implementing a change. It is a thin prevention layer, not a general coding methodology. It applies the same catalog to the implementation decisions and resulting diff of the current task. Do not turn it into a repository-wide cleanup unless the user asks.

**Before coding — work from the repository inward:**
1. Search existing helpers, utilities, sibling implementations, base classes, validators, and installed dependencies before creating a new one.
2. Trace the real types, invariants, and error behavior on the path being changed. Do not compensate for unknowns with probing chains or broad fallbacks.
3. Identify the scope: exploratory code, an internal module, or a stable/public boundary. Look for documented migrations, compatibility requirements, and actual plugin loading before judging their shapes.

**While coding — use five decision rules:**
- **Reuse before recreating** (patterns 1–3): extend or call what the repository already owns; do not fork helpers or standard-library behavior.
- **Concrete before abstract** (patterns 4–7): keep a local need local; wait for a real second implementation before adding an abstraction, registry, global knob, or public API.
- **Know before defending** (patterns 8–11): write against the actual contract; defensive flexibility belongs at real trust boundaries, not wherever the type was left unexamined.
- **Explain why, not what** (patterns 12–14): comments record constraints and non-obvious reasons; remove narration, boilerplate, and implementation debris.
- **Test behavior, not scaffolding** (patterns 15–16): assert externally meaningful behavior and failure cases, not merely that mocks were called.

**After coding — audit the diff, index the repo:**
1. Run the relevant tests and inspect only the current diff against the catalog. Duplication checks still search the whole repository.
2. Fix safe slop introduced by the current change. Do not opportunistically refactor unrelated pre-existing findings; report them separately.
3. Record justified exemptions briefly. Expand into a full findings table only when meaningful debt or behavior risk remains.
4. End with a compact guard audit, for example: reused existing helper; no new speculative API; one exploratory exemption; tests passed.

### Exploration is not production debt

Exploration may justify temporary duplication, hard-coded values, parallel variants, or rough scripts, especially under `experiments/`, `scratch/`, `notebooks/`, and `prototypes/`. Treat these as severity 1 when they are contained and deliberate. The exemption ends when the code moves into a core package, stable API, shared module, or merge-ready path. Guard mode should preserve room to discover; hardening is where temporary branches converge and the normal catalog applies.

**Scoping:** for a PR/diff, scan only changed files but check duplication against the *whole* repo. For a whole repo, go module by module; propose the order and let the maintainer prune. In Guard mode, constrain edits and the post-pass to the current task's diff unless broader cleanup was requested.

## Common mistakes (seen in the wild)

| Mistake | Reality |
|---|---|
| "This error type is clearly accidental — I'll improve it" | That's a behavior change. Callers catch specific exceptions. Report it. |
| Deleting an "unused" helper that scripts/templates/reflection use | Grep everything, including non-code files, before pattern-5 deletions. |
| One mega-commit fixing 9 patterns | Un-reviewable and un-revertable. One pattern-class per commit. |
| Fixing before running the suite once | You can't tell "I broke it" from "it was broken." Oracle first. |
| Rewriting working defensive code at a trust boundary | Untrusted input is the one place paranoia is correct. |
| Treating the catalog as a checklist to maximize | The goal is a healthier repo, not a body count. Severity 1 findings stay. |

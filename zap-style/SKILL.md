---
name: zap-style
description: Zap house C#/TypeScript style for the BI codebase, distilled from Simon Oxtoby's and Robert Macdonald's commits. Scores a change on a weighted 7-axis rubric and lists the material findings a reviewer would raise; loop until FINDINGS is 0. Apply when WRITING, REVIEWING, or SCORING Zap C#/TypeScript so the code survives review unchanged. Triggers on authoring new BI code, editing existing BI code, reviewing a BI diff, or asking for a style score.
---

# zap-style — write BI code the team won't rewrite

## Intent

This skill encodes the house C#/TypeScript style of the BI codebase, distilled from the commits of its most exacting authors. The goal is narrow: produce code that **would survive review unchanged** — that Simon Oxtoby or Robert Macdonald would not rewrite. The example library is the spec; the prose only tells you how to read and weight it, and how to score against it. Throwaway, demo, and hack code is scored like any other — "it's only a hack" exempts nothing.

It works in two modes, both driven by the same rubric:
- **Author / review** — write or edit BI code so it matches house style first time.
- **Score** — grade a change 0-10 against the rubric, emit the exact failing criteria, and (in a loop) re-score after fixes until it clears the bar.

### Principles
- **CLEAN (Uncle Bob).** Intent-named symbols, small single-responsibility functions, zero dead code. Name the boolean, the predicate, the method — components and `if`s should read as sentences (`canUseActivePaneContext`, `DiagnoseQueryErrors`, `ShouldRetryAfterFailure`).
- **Minimum-information diff — minimise the change set's information content.** Change only what the change needs; of two correct versions, ship the one a reader could reconstruct from the shorter description — fewest new names, types, files, and concepts. Preserve surrounding idioms — keep `?.`, keep the existing helper family, keep the brace-free ladder. Don't reformat or rename on the side; a pure typo/reformat lands in its **own** honestly-titled commit.
- **Surgical / smallest blast radius.** A two-line guard can fix a whole bug — prefer it to a refactor. **Extend or reuse an existing helper rather than forking a parallel one.** Push deps to the constructor so call sites shed args; when you delete code, delete it *everywhere* including `.csproj`.
- **Grouped change.** Multi-file edits stay in one folder / clearly grouped; a rename touches only the symbol, its refs, and the `.csproj` move.
- **Modern idiom.** Pattern matching over `if`-ladders and `IfIs<T>(lambda)`; records + `init` over mutable classes; switch expressions for dispatch; `?? throw` / `_ => throw` to fail loudly; collection expressions (`[]`, `[..xs, x]`).
- **Self-documenting; comments are a last resort.** Intent-named symbols carry the meaning — default to **zero** comments. Write one ONLY where CLEAN code physically cannot encode the *why*: a non-obvious constraint, a counter-intuitive workaround. Then it's a short trailing why-line, never a banner or `<summary>`. The *why* of the change lives in the commit message, not the source.

### How to read & weight the examples
- **Prose is the frame; the examples carry the weight.** When in doubt, match the closest example.
- **Nothing here decides it — no close example, no principle, just a stylistic call? Interpolate, do not invent.** Infer the aesthetic the nearest examples share — terse, intent-named, minimal-diff, fail-loud — and pick what they would imply. Between two equally defensible readings, take the terser.
- Each example: a one-line caption (the WHY) + tight code + axis tags `[C#]`/`[TS]`, optional `[learning]`/`[learning-proxy]`/`[domain:*]`, and the source commit sha. ❌ = anti-pattern (before) → ✅ = house idiom (after).
- **`[learning]` examples OUTWEIGH plain ones.** They are cases where the model wrote code *through this very skill* and a senior still changed it — the highest-signal corrections, flagged ⭐. `[learning-proxy]` = a change a reviewer demanded in PR review (next-highest signal). If a learning example conflicts with a plain one, follow the learning.

---

# The example library (loaded on demand)

The examples live in `references/`. Load only the shards the change touches — do not read all of them every pass. There are two kinds:

**Concern shards** (cross-cutting idioms — apply to *all* BI code in that language):
- `references/csharp-core.md` — nullability & null-flow · pattern matching & switch dispatch · fail-loudly & exceptions · types & immutability (records) · LINQ & intent-named behaviour
- `references/csharp-diff.md` — diff shape · surgical change · smallest blast radius (the heaviest, highest-signal axis)
- `references/react-ts.md` — React component structure · rendering & dispatch · controllers/models/event-reduce · TypeScript types & narrowing
- `references/tests.md` — test shape, fakes-over-mocks, EasyAssertions vocabulary

**Domain shards** (area-specific vocabulary & conventions — load the one for the subsystem the change is in):
- `references/modeller.md` — data-prep backend that *writes* the cube: pipeline steps, updaters, transforms, jobs, loaders, warehouse, state objects, DI. The signature idiom is `Match` over a closed union.
- `references/bi.md` — analytics side that *reads* the cube: MDX generation via the typed DOM, `Dimensionality`, UniqueName identity, CellSet/result rendering.

**Which to load:** for a C# change, always load both concern shards (`csharp-core`, `csharp-diff`); add `tests.md` if tests are in scope; add the domain shard for the subsystem — `Modeller/**` (cube authoring, warehouse, ETL) → `modeller.md`; analytics/query code that generates or runs MDX or renders results → `bi.md`. For a TS/React change, load `react-ts.md` (+ `tests.md`). Never load shards for languages the change doesn't touch.

---

# Scoring rubric

The score is **derived from a checklist, not a vibe**, and scoring is **strict and discrete** so it is reproducible and terminating.

The 0–10 is a display. The thing the loop chases is the **count of material findings**: a reproducible, selective signal where the bare number was noisy.

## Two things are assessed separately

- **Style score (0–10)** — how house-idiomatic the *code* is. The loop bar, in **both** author and review mode, is **zero material findings**.
- **Diff-discipline gate (both modes)** — whether the *commit/diff* is well-organized. Reported pass/fail **alongside** the score; it **never moves the 0–10**.

> Why the split: calibrating against the source authors' own commits, *every* sub-9.5 was a diff-hygiene issue (bundled cleanup, trailing whitespace), never unidiomatic code. Folding commit organization into the code-style number made good code look bad and made the loop jitter. Keep them apart.

## Style axes and weights

| # | Axis | Weight | Critical? |
|---|------|:------:|:---------:|
| 1 | **Reuse & information economy** | 1.5 | ✅ |
| 2 | **Surgical / blast radius** | 2.0 | ✅ |
| 3 | **Modern idiom** | 1.5 | |
| 4 | **CLEAN naming & structure** | 1.5 | |
| 5 | **Nullability & fail-loud** | 1.5 | ✅ |
| 6 | **Self-documenting (comments)** | 1.0 | |
| 7 | **Tests** | 1.0 | |

## Scoring rules

- **Discrete only.** An axis is scored **1 (Pass) / 0.5 (Partial) / 0 (Fail)** — never 0.9, never a "soft" fraction.
- **Findings drive everything — an axis score is a pure function of its material findings.** 0 findings → **1.0**. 1–2 findings → **0.5**. **≥3 findings → 0** (density escalation: repeated instances of the same issue compound — count *each* instance, never average them away). Exception: a single finding that matches a **Named Fail** → **0** immediately. The Named Fails are **exhaustive — exactly four**: forked parallel abstraction (axis 1) · dead code/csproj left behind (axis 2) · god-function (axis 4) · swallowed error or bug-hiding null-annotation (axis 5). Axes 3, 6 and 7 have no Named Fail — they reach 0 only by count (≥3). An axis may drop below 1.0 **only** on a cited `file:line` finding — no soft deductions, ever.
- **One axis per finding.** File each finding under exactly one axis — the most specific one — and never double-count a single root cause across axes (every instance of one duplicated block → Reuse; a swallowed catch is Null even when it lives in a test file; dead code is always Surgical, never CLEAN).
- **Score the change, not the neighbourhood.** Findings attach only to lines/constructs the diff **adds, modifies, or orphans** — code and `.csproj` entries made dead *by* this diff are part of the change, and deleting them is not "refactoring untouched code". Pre-existing issues in code the diff merely touches are noted as NITs, not scored — with one exception: a genuine *correctness bug* in the very logic being modified is material, filed under axis 2 (axis 5 when the bug is null/error-flow) — the "fix the hidden bug, don't just tidy around it" lesson (csharp-core learning-proxy dccbe7d). This also bounds `TO FIX` items: a fix must never demand refactoring untouched code.
- **Nit-floor — the boundary rule.** A finding is **material only if a reviewer would leave a comment asking for the change.** If a formatter/linter would fix it, or a reviewer would fix it silently *without* comment, it is a **NIT** — list it under `NITS:`, never score it, never count it. NITS, enumerated (do not score any of these):
  - trailing / leading / blank-line whitespace; final-newline present or absent
  - import ordering and grouping
  - brace style, same-line vs next-line, `{}` vs `{ }`, `;;`, redundant parens
  - reordering of independent members with no semantic effect
  - clear-but-verbose names (any count) where the meaning is already obvious — naming is material only when opaque or misleading
  - anything a **whitespace/formatting-only** pass fixes (`prettier`, `dotnet format whitespace`) — this does NOT cover style-analyzer rewrites (if-ladder→pattern matching, collection expressions, target-typed `new`): those are axis-3 material, not nits
  - genuinely tool-generated files (`*.Designer.cs`, `*.Generated.cs`, resgen/typing mirrors, `.resx` output) — excluded from scoring entirely, not even NITs. Exclusion requires the file to actually be generated — a generated-code header **and** an evident generator/source (a pasted header alone doesn't qualify); hand-written hunks hiding in a "generated"-named file are scored normally
- **Critical axes (1, 2, 5):** a *Fail* on any of these zeroes an axis of weight ≥ 1.5, so the computed score necessarily lands at **≤ 8.5** — no separate cap exists or is needed.

## Per-axis criteria (1 / 0.5 / 0)

1. **Reuse & information economy** — reuses an existing helper/family instead of forking a parallel one; introduces the fewest new names/types/concepts the change needs. Named *Fail*: adds a parallel abstraction where one existed. (Bundling, reformat and whitespace are **not** here — they're the diff-discipline gate.)
2. **Surgical / blast radius** — smallest edit that fixes the thing (a guard over a refactor); deps pushed to the constructor so call sites shed args; deleted code removed *everywhere* including the `.csproj` `<Compile Include>`. Named *Fail*: dead code or csproj entries left behind. (Whether the change is appropriately *scoped* — one concern, not broader than the problem — is the diff-discipline gate's job, not this axis. Score only the mechanics of the edits that are here.)
3. **Modern idiom** — pattern matching / switch expressions over `if`-ladders; records + `init`; collection expressions; `?? throw` / `_ => throw`; (TS) `reactive(named fn)`, event-reduce, React-19 idioms; (modeller) `Match` over a closed union. Each outdated construct where the modern form is house-standard is a material finding. (No Named Fail — this axis reaches 0 only by count.)
4. **CLEAN naming & structure** — intent-named symbols; conditions/getters named so the call site reads as a sentence; small single-responsibility functions. Named *Fail*: a god-function. Opaque or misleading names are material findings, not instant Fails. (A clear-but-verbose name is a NIT; dead code belongs to axis 2, not here.)
5. **Nullability & fail-loud** — house null-flow idioms (`?.`, `?? fallback`, `.NotNull()`, `[NotNullWhen]`); fails loudly at trust boundaries; no swallowed exceptions. Named *Fail*: a swallowed error, or a null-annotation/check that hides a bug (including a non-null contract the body provably violates). Correctly *removing* a false throw or needless fallback is a Pass, not a violation.
6. **Self-documenting (comments)** — zero comments unless a genuine why-line is required; no `<summary>` essays, no comment restating the code. One decision owns one why-line: the same why stated twice — even in different files or layers — is a finding. Each explanatory comment that CLEAN code should have encoded is a material finding. (No Named Fail — this axis reaches 0 only by count.)
7. **Tests** — `Method_Condition_ExpectedResult`; AAA split by blank lines; locals `sut`/`result`; hand-written fakes over `Substitute.For` + `.Returns`; EasyAssertions collection vocabulary (`.ShouldBeEmpty()`, `.ShouldBeASingular<T>()`) not LINQ-into-`ShouldBe`.

## Computing the style score

1. **Scope.** Identify languages + subsystem; load the matching shards — load **every** domain shard the diff touches, not just one. **Axes 1–6 are in scope for every diff containing hand-written code; Tests (axis 7) is the only conditional axis** — n/a when the diff has no test surface. (Language-specific criteria within an axis simply don't apply across languages: TS idioms are irrelevant to a pure C# diff and vice versa — the axis itself stays in scope.) Exclude generated files before scoring. If nothing hand-written is in scope, report `SCORE: n/a — no scorable surface` and `FINDINGS: 0`.
2. **Find.** List every *material* finding (nit-floor applied) with `file:line`, each filed under exactly one axis. Count them.
3. **Score axes** from their finding counts (mapping in Scoring rules).
4. **Compute:** `score = 10 × Σ(axis_score × weight) / Σ(weight)` over in-scope axes only, rounded to one decimal. With all 7 axes in scope, Σweight = **10.0**. Worked example: one Comments finding, Tests n/a, everything else clean → 10 × 8.5 / 9.0 = **9.4**.
5. **Bars — one signal.** `FINDINGS = 0` ⇒ every in-scope axis is 1.0 ⇒ score is **exactly 10.0**, by definition. Any material finding ⇒ score **≤ 9.5**. So "loop until score > 9.5" and "loop until zero findings" are the *same* bar.

## Diff-discipline gate (both modes)

A pass/fail checklist on the *diff*. It runs in author mode too — a freshly authored change *should* be single-concern by construction, and the gate is the check that it is:
- single coherent concern — no unrelated tidy-ups **and no second code concern** bundled in (e.g. a modernization sweep folded into a bug fix, or a nullability sweep carrying a typo-rename);
- no drive-by rename/reformat (a pure typo/reformat belongs in its own honestly-titled commit);
- no incidental whitespace on touched or blank lines.

Report `DIFF DISCIPLINE: pass`, or list each violation as a `SPLIT:` item with `file:line` and the split it needs. `SPLIT:` items carry to the next loop pass alongside `TO FIX`, and **the loop does not terminate while the gate fails.** A split-off commit is not an exit hatch: **each commit produced by a SPLIT becomes its own scoring subject**, and the work as a whole is done only when every one of them has passed (or been explicitly discarded).

## Output contract

A run with findings:

```
SCORE: 9.4 / 10   (Reuse 1.0 · Surgical 1.0 · Idiom 1.0 · CLEAN 1.0 · Null 1.0 · Comments 0.5 · Tests n/a)
FINDINGS: 1 material
TO FIX (each line = one material finding):
  • [Comments] Foo.cs:42 — doc-comment restates the code; cut it, why goes in the commit. (ref: csharp-diff — "the why lives in the commit message")
NITS (not scored): Foo.cs:60 trailing whitespace; imports reordered
DIFF DISCIPLINE: FAIL — SPLIT: Bar.cs:88 assertion cleanup bundled into a parameter-removal; land it in its own commit.
```

A passing run:

```
SCORE: 10 / 10
FINDINGS: 0
DIFF DISCIPLINE: pass
```

- `TO FIX` and `SPLIT` items are the **only** feedback that carries to the next loop pass. A `TO FIX` line = axis tag · `file:line` · the fix · a ref (shard name + the example's caption, quoted or closely paraphrased — not a section anchor).

## Loop protocol

Each pass, to avoid lost-in-the-middle drift, the model reads **only**: this SKILL.md (rubric + contract) + the shards the change touches + the previous pass's `TO FIX` and `SPLIT` items. It does **not** re-read the whole example library or the prior transcript.

1. Score the current change; emit the output contract (including `FINDINGS: N` and the gate line).
2. If **`FINDINGS = 0` and `DIFF DISCIPLINE: pass`** → **stop**, report the final score. (Don't chase decimals — zero findings with a clean gate *is* the bar.)
3. Otherwise apply the `TO FIX` and `SPLIT` items as a literal change spec — fixing only those, surgically — then re-score.

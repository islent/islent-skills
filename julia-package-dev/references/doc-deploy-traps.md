# Documenter build traps (non-obvious only)

> The **build-side** traps the official manual will never tell you — the
> misleading error strings and the verified fixes.
> Every rule is **general to any Julia package**.
> Official docs are **linked, not restated**: [Documenter manual](https://documenter.juliadocs.org/stable/),
> [1.0 migration guide](https://documenter.juliadocs.org/stable/man/guide/).
> Deployment-side traps (Pages 404, `DOCUMENTER_KEY`) are in
> [`github-pages-and-deploy-auth.md`](github-pages-and-deploy-auth.md).

---

## Symptom → cause → fix

Error strings are quoted verbatim so you can grep for them; the explanation
lives in the section named in the last column.

| Symptom | Cause | Fix |
|---|---|---|
| `makedocs encountered errors ([:docs_block, :missing_docs])` | 1.x flipped the `checkdocs` default to `:all` | `checkdocs = :none` → [checkdocs](#checkdocs-none-not-exports) |
| `Unrecognised asset class` (`HTMLWriter.jl:112`) | 1.x `HTML(assets=…)` accepts only `ico`/`css`/`js` | keep those three → [assets](#assets-1x-accepts-only-three-classes) |
| build green, favicon/asset 404s | `assets` validates the **class**, not the **existence** | diff against `docs/src/assets` → [assets](#assets-1x-accepts-only-three-classes) |
| `Package <Pkg> [...] is required but does not seem to be installed` | the docs env has **no `Manifest.toml`** — *not* "the package is missing" | instantiate the docs env → [docs env](#the-docs-environment-is-separate) |
| `Package <Pkg> not found in current path` inside `@repl`/`@example` | not a **direct** dep of `docs/Project.toml` | add it as a direct dep → [@repl / @example](#repl--example) |
| `no docs found for 'X' in @docs block` | the binding has **no docstring of its own** | delete it, or write the docstring → [name resolution](#docs-name-resolution) |
| `undefined binding 'X' in @docs block` | `X` is documented but **not exported** | use `MyPkg.X` → [name resolution](#docs-name-resolution) |
| `duplicate docs found for 'X' in @docs block` | the same docstring rendered on two pages | keep one canonical `@docs` block |
| `Cannot resolve @ref` / `no doc found for reference: '[X](@ref)'` | the target has no anchor in this manual | three different fixes → [@ref](#broken-ref--three-categories-three-different-fixes) |
| `Cannot load <Sibling> into <Pkg>` (local only) | the **stale local `docs/Manifest.toml`** pinned a sibling to a version missing a dep | re-resolve; do not chase it |
| exit code 0 but the HTML holds `UndefVarError`/`MethodError`/`ERROR:` | `@repl`/`@example` failures are often **not fatal** | scan the built HTML → [verification](#verification) |
| build hangs until the CI timeout | an `@example` spawned workers the main process then waits on | demote to a plain fence → [@repl / @example](#repl--example) |
| SSH/Pages symptoms (`Permission denied (publickey)`, 404, `Deployment failed, try again later`) | deployment-side, not build-side | → [`github-pages-and-deploy-auth.md`](github-pages-and-deploy-auth.md) |

### `checkdocs`: `:none`, **not** `:exports`

- **1.x defaults to `:all`** ⇒ every docstring not in a canonical `@docs` /
  `@autodocs` block is a hard error. Fix: `checkdocs = :none`. Not a cop-out: a
  `@docs` block naming a binding that no longer exists **still errors** — the
  real regression check. It only silences "docstrings not in the manual".
- **`:exports` is NOT a drop-in substitute**: it still flags the **module's own
  docstring** (`names(MyPkg)` contains `MyPkg`) and every docstring inside an
  **exported submodule** (the check recurses). **Do not "fix" `:all` by writing
  thirty `@docs` blocks** — that is a documentation project, not a build fix.
- **The `$(EXPORTS)` trap (DocStringExtensions).** A module docstring ending in
  `$(EXPORTS)` expands into a Markdown list of `[`name`](@ref)` links for *every*
  exported name — including re-exported third-party ones (`norm`, `median`,
  `value`, `@u_str`). Rendering it forces Documenter to resolve hundreds of
  `@ref`s that can never have an anchor in your manual ⇒ one `@docs MyPkg`
  detonates with `:cross_references`. Fix: drop `$(EXPORTS)`, or never reference
  the module docstring.

### `@docs` name resolution

| Situation | What Documenter says | Fix |
|---|---|---|
| exported **and** documented | *(resolves)* | keep the bare name |
| **not exported**, has a docstring | `undefined binding 'X' in @docs block` | qualified name `MyPkg.X` |
| no docstring — including a method extending `Base.run` / `Base.step` / `Base.length` | `no docs found for 'X' in @docs block` | delete from the `@docs` block (or write the docstring) |

The **`Base.X` case is the nastiest**: the docstring *is* in your source, but
it is attached to the **`Base` binding**, not to your module's binding, so
Documenter's module-scoped lookup cannot find it. This is a **heuristic, not a
law** ⇒ never guess; **measure** with the oracle (see
[verification](#verification)).

### Broken `@ref` — three categories, three different fixes

1. **Page-level reference** (a link to a page title) ⇒ use a **relative `.md`
   link** instead of `@ref`: `[KDK](api/KDK.md)`. Documenter rewrites the
   target to the right URL *and* validates that the file exists — deterministic,
   unlike title matching.
2. **Cross-package or undocumented function** ⇒ it has no anchor in *this*
   manual ⇒ **degrade to inline code**: `` `select_backend` `` instead of
   `` [`select_backend`](@ref) ``.
3. **A `@ref` inside a *source docstring*** ⇒ the worst case: Documenter
   attributes the error to the **page that renders the docstring**, not to the
   `.jl` file. When an `@ref` error names a page you never touched, grep the
   sources: `Select-String -Path src/**/*.jl -Pattern '\]\(@ref\)'`.

### `assets`: 1.x accepts only three classes

- 1.x classifies each entry and errors outside `[:ico, :css, :js]` ⇒
  `HTMLWriter.jl:112 → error("Unrecognised asset class ...")`. In 0.26/0.27 the
  list meant "copy these files verbatim", so old lists of `.png` / `.pdf` /
  `.svg` / `.html` used to pass.
- Everything else under `docs/src` is auto-copied (`SetupBuildDirectory`) ⇒
  `![](assets/fig.png)` keeps working without being registered.
- **A wrong filename is a silent 404**, not a build error: the class is
  validated, the existence is not ⇒ green build, deployed site, missing favicon
  forever.
- **Diagnostic rule:** if the error points at `HTMLWriter.jl` / `format=`, stop
  looking at `@docs` — an independent failure axis.

### The docs environment is separate

- `docs/` is its own environment: a `dev` install in your default env is
  **invisible** to `--project=docs`, which sees only what
  `docs/Project.toml` + `docs/Manifest.toml` resolve.
- `Package X is required but does not seem to be installed` means **"this
  environment has no `Manifest.toml`"**, *not* "the package is missing from the
  depot". 30-second falsification: a temp env with only `[deps] Documenter`
  and no Manifest gives the identical error.
- `Pkg.develop(PackageSpec(; path=pwd()))` mutates **both**
  `docs/Manifest.toml` (gitignored, fine) and the git-tracked
  `docs/Project.toml`, appending a machine-specific absolute path
  (`[sources] MyPkg = {path = "E:\\…"}`). `[sources]` belongs in the Manifest —
  revert before committing with `git checkout HEAD -- docs/Project.toml`.

### `@repl` / `@example`

1. **They run in the docs env, where only *direct* deps are importable.** A
   merely transitive package is not loadable ⇒ `Package X not found in current
   path`. Add every package the blocks `using` as a **direct** dep of
   `docs/Project.toml`; and `make.jl` must `using` every package the docs import.
2. **Failures are often not fatal.** The output is rendered into the page and
   the build still exits 0 ⇒ a green build can ship `UndefVarError` /
   `MethodError` blocks. Exit code 0 is **not** proof.
3. **They really execute.** The first build compiles/precompiles everything ⇒
   **minutes**, not seconds. Budget for it; do not "optimise" by deleting the
   example. To *show* code without evaluating it, use a plain fence with the
   language tag plus `(raw)`. A block that calls `addprocs` and then
   distributed code makes Documenter's main process the master of a worker pool
   ⇒ if the workers fail to precompile, `remotecall_eval` never returns and the
   **whole build hangs** until the CI timeout. Demote it to a plain fence; set
   `timeout-minutes: 30`.

---

## The local-build-does-not-reproduce trap

**The single most expensive mistake in this whole area.** The symptom is always
the same shape: *"CI fails, my machine is green"* — and it is almost never a
code problem.

**Why:** `docs/Manifest.toml` is **gitignored**, so the file on your disk is a
snapshot from whenever you last resolved, and it pins **Documenter itself**. In
the observed case it was Documenter 0.26 locally while CI resolved 1.19 ⇒ the
local build ran a *different Documenter with different defaults* and the CI
failure could not be reproduced at all. A stale manifest can also produce a
**local-only** failure unrelated to CI — e.g. `Cannot load <Sibling> into
<Pkg>`, where the old manifest pinned a sibling to a version that did not
declare that dependency.

⇒ **Everything else in this file is unverifiable until your local Documenter
version matches CI's.**

```powershell
cd E:\path\to\MyPkg.jl                    # the PACKAGE root, not docs/

# 1. Back up + remove the stale Manifest (it is gitignored; nothing is lost)
Copy-Item docs\Manifest.toml docs\Manifest.toml.bak -Force
Remove-Item docs\Manifest.toml

# 2. Re-resolve exactly like CI does
julia --project=docs -e '
  using Pkg;
  Pkg.add("Documenter");
  Pkg.develop(PackageSpec(; path=pwd()));
  Pkg.instantiate();'

# 3. Build
julia --project=docs --color=yes docs/make.jl
```

Check the resolved Documenter version *before* debugging anything else:
`Select-String -Path docs\Manifest.toml -Pattern 'Documenter'`.

**Rules that follow from it:**

1. **Put the environment back the way you found it** (restore
   `docs/Manifest.toml` and `docs/Project.toml`) so the next person does not
   inherit a poisoned environment — and never commit the `[sources]` path.
2. **A local failure that comes from a stale Manifest is not evidence about
   CI.** Re-resolve first, then compare.
3. **Precompile contention bites when two doc builds run in parallel**: one
   blocks silently on the other's precompile lock and can look like a hang. Run
   them sequentially.

---

## Documenter 1.x migration checklist

1. **Re-resolve the docs environment from scratch.** Everything below is
   unverifiable until the local Documenter version matches CI's.
2. **`checkdocs = :none`** — not `:exports`.
3. **Audit `HTML(assets=…)`**: keep only `ico` / `css` / `js`; verify each
   remaining entry **exists on disk**.
4. **Never `@docs` a module docstring containing `$(EXPORTS)`.**
5. **Sweep every `@docs` name** through the oracle and bucket each one: keep
   bare / qualify / delete.
6. **De-duplicate docstrings across pages** — one canonical `@docs` block per
   binding; narrative pages point at it.
7. **Replace broken `@ref`s** — page-level ⇒ relative `.md` links; cross-package
   or undocumented functions ⇒ inline code. **Grep `src/**/*.jl` for
   `](@ref)`** too.
8. **Make `docs/Project.toml` list every package the `@repl` / `@example` blocks
   `using`** — direct deps only, UUIDs copied from upstream. And `make.jl` must
   `using` every package the docs import.
9. **`strict = false` → `warnonly`**, choosing deliberately which categories are
   non-fatal. `warnonly` does **not** cover `:example_block`. Watch out for an
   ARGS-gated `warnonly = ("warnonly=true" in ARGS) || ...` — it is lenient
   locally and **strict in CI** unless the workflow passes the flag. Never
   validate CI behaviour with a lenient local build.
10. **Move distributed / worker-spawning code out of `@example` blocks**; set
    `timeout-minutes: 30`.
11. **Scan the built HTML** for `UndefVarError` / `MethodError` / `ERROR:`
    before declaring victory.

---

## Verification

### The Documenter oracle — every broken `@docs` name in one pass

Do **not** fix `@docs` names one at a time through full builds. Collect every
name currently in an `@docs` block onto one scratch page and let Documenter
say which resolve. It is fast (no `@example`, no images, no plotting).

The script has three parts: **(1)** grep every ```` ```@docs ```` block in
`docs/src/**/*.md` for the names it lists; **(2)** write them all into a single
scratch page in a `mktempdir()`; **(3)** `makedocs` that scratch project with
`modules = [MyPkg]`, `checkdocs = :none`, `warnonly = :missing_docs`,
`doctest = false`, `remotes = nothing`. Run it **twice** — bare names and
`MyPkg.`-qualified names — then read the log for
`no docs found for|undefined binding`, and bucket each name per the
[name-resolution](#docs-name-resolution) table.

**Traps in the oracle script itself:** `build` must be **relative to `root`**
(an absolute path nests `build/build/build/…` until `ENAMETOOLONG`); `source`
must **not** point at `root` itself; `remotes = nothing` is required for a
scratch project outside a package; keep `warnonly = :missing_docs` so
`:docs_block` stays fatal and the run reports **every** bad name in one pass
instead of stopping at the first.

### Prove the rendered pages contain no errors

Exit code 0 is not enough — `@repl`/`@example` failures render as ERROR
blocks without failing the build:

```powershell
python -c "import re,glob,sys; [print(f) for f in glob.glob('docs/build/**/index.html', recursive=True) if re.search(r'UndefVarError|MethodError|ERROR:', open(f, encoding='utf-8', errors='ignore').read())]"
```

Empty output is the pass condition. A **legitimate** `ERROR:` block can exist
(a page deliberately demonstrating an unsupported input) — know which ones are
intentional before "fixing" them.

### Cheap sanity checks

Four greps answer most of it: which `Documenter` version the docs env
resolved; whether `docs/Project.toml` has a leaked `[sources]` absolute path;
whether every `assets` entry exists on disk; whether any `src/**/*.jl`
docstring still contains `](@ref)`.

### Definition of "done"

- `julia --project=docs docs/make.jl` exits **0** and the build log contains
  **no `┌ Error:` block** (benign warnings such as `overwriting build/assets`
  or `could not auto-detect the building environment` are fine);
- the HTML scan prints **nothing**;
- `git status` shows **no** `docs/Project.toml` / `docs/Manifest.toml` churn
  and no `[sources]` path.

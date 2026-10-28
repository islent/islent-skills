# CI, test layout & coverage (non-obvious only)

> The traps behind `SKILL.md` §4 that a "the workflow is green" reading will not
> catch — resolution surprises, gates that silently do not fire, and artefacts
> that lie about what ran.
> Every rule is **general to any Julia package**.
> Official docs are **linked, not restated**: [workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) · [setup-julia](https://github.com/julia-actions/setup-julia) · [Pkg.test](https://pkgdocs.julialang.org/dev/api/#Pkg.test) · [codecov](https://docs.codecov.com/).

---

## Sibling packages resolve from the registry, not your tree

`Pkg.test("MyPkg")` in the package's own env resolves `MyPkg`'s dependencies
**from the registry** — the *released* siblings, not your working tree. A CI job
that must test against local siblings needs its own environment:

```julia
using Pkg
Pkg.develop([PackageSpec(path="."), PackageSpec(path="../Sibling.jl")])
```

`pkg"dev ."` alone develops only the package under test (`.` is the checkout
root), so the siblings still come from the registry.

**`monorepo/` is not a committed directory.** Julia creates
`monorepo/Project.toml` + `Manifest.toml` on the first write (the first write
*is* `pkg"dev ."`). It is usually **not** gitignored either — reproducing CI
locally leaves an untracked `monorepo/` in your tree.

**The `shell: julia --project=monorepo {0}` trick.** `{0}` is GitHub's
placeholder for a **temporary script file** holding the step's `run:` body;
setting `shell:` that way makes GitHub run the body as a **Julia script**
instead of bash. Hence the body is Julia, not shell — no `cd`, `&&` or `$VAR` in
such a step:

```yaml
- name: Install Julia dependencies
  shell: julia --project=monorepo {0}
  run: |
    using Pkg;
    pkg"dev ."
```

`pkg"..."` is [Pkg REPL-mode string syntax](https://pkgdocs.julialang.org/dev/api/#Pkg.REPLMode)
(Julia ≥ 1.9), **not** a shell macro — on ≥ 1.9 it prints
`Warning: The Pkg REPL mode is intended for interactive use only...` to stderr,
which is **normal in the CI log, not a failure**.

---

## A committed `[sources]` path is a CI-only hard failure

A committed test env may pin a sibling by path — `test/Project.toml` with
`[sources] Sibling = {path = "../../Sibling.jl"}`. Such paths resolve against
the *project file's* directory, so `../../Sibling.jl` = **`<repo-parent>/Sibling.jl`**,
a sibling checkout that exists only in a multi-repo local layout.

**On a runner there is no sibling checkout**, and `Pkg.test` does **not** fall
back to the registry:

```
ERROR: expected package `CSV [336ed68f]` to exist at path `.../CSV.jl`

This package is referenced in the manifest file: .../jl_gsWfON/Manifest.toml
It is required by: MyPkg
```

Pkg *does* have a fallback (`drop_external_path_sources!`), but it fires only
when the package under test was **installed** (has a `tree_hash`), **not** when
it was `dev`'d from a path — and CI always `dev`s the package. **The fallback
is unavailable exactly where it would be needed.** ⇒ **A relative `[sources]`
path that escapes the repo root is a CI-only hard failure**, and with
`continue-on-error: true` it will not even turn the job red.

⇒ **The CI-safe committed form is `url` + `rev`** — `[sources] Sibling =
{url = "https://github.com/OWNER/Sibling.jl", rev = "main"}`. A `url` source
resolves identically on a runner and on a workstation, and it is the only way
to depend on a package that is **not in the registry**. Rule of thumb: `url`
for unregistered deps; `path` only for a path guaranteed to exist **inside**
the checkout. An **absolute** path breaks every non-Windows machine
(`SKILL.md` §6.4).

---

## `continue-on-error: true`: a green job can hide failed tests

`continue-on-error` is a property of the **step**; the job's conclusion is the
OR of its steps *with the soft-failed ones excluded*. A failing test step
therefore leaves the **job green** — the UI shows a yellow warning triangle on
the step while the run summary still says success.

⇒ **Read the step conclusions, not the job or the run:**
`GET /repos/OWNER/REPO/actions/runs/RUNID/jobs` → `.jobs.steps[].conclusion`.

⇒ **Know which mode you are in before trusting a tick.** If you take over a
repo with this flag, the first useful change is to remove it and fix the
resulting red — **separately** from feature work, so a pre-existing failure is
distinguishable from one you introduced.

---

## Commit-message gates are inert on pull requests

**`github.event.head_commit` is a field of the `push` payload.** The
`pull_request` payload has no such top-level field, and neither does
`workflow_dispatch`. So `github.event.head_commit.message` evaluates to `null`
there and `contains(null, '[skip ci]')` is `false` — **every commit-message
gate is inert on PRs and only fires on `push`.**

- **Markers cannot be used to control PR CI.** `[skip ci]` does not skip CI on
  a pull request; it only suppresses the `push` trigger. To keep a PR quiet you
  need a branch-protection rule, not a commit marker.
- **A `[compat]`-gated apply workflow is unreachable via `workflow_dispatch`** —
  the job is always skipped. If the gated workflow is the only entry point
  there is no working manual path: dispatch it elsewhere or drop the `if:`.
- **Do not assume a marker caused a red PR.** Read the failing step, not the
  `if:` line.
- **Real-world commit messages often use the bare word `compat:`** and never
  the bracketed `[compat]` token the gate tests for — that gate fires only when
  a human writes the bracketed token by hand.

⇒ **Rule:** mark every commit, use `[skip ci]` for doc/comment-only changes,
use the *bracketed* `[compat]` token for the compat-bump merge commit — and
**never rely on markers to control PR CI**.

---

## Plotting packages need an X server

Without the `apt-get` install of the X libraries and `xvfb-run` around the
test command, any test that plots dies with a GLFW error at `using GLMakie`
load time — **before any test body runs**, so you get *zero* test results, not
"the plotting tests failed". The explicit screen geometry matters too:
`xvfb-run`'s default (`-screen 0 640x480x8`) is **too small for GLMakie** and
the wrong colour depth.

⇒ **The X-server block is required by the plotting dependency, not by "CI".**
Check whether the package really pulls in `GLMakie`; if not, use the plain
`julia-actions/julia-runtest@v1` shape and skip the whole apparatus.

⇒ **A genuinely headless package should not need them at all.** The General
registry's AutoMerge runs `import <Pkg>` on a headless runner, so a *hard*
`GLMakie` dependency means merely loading the package needs a display and the
registry rejects you. The escape hatch is `weakdeps` + a package extension, so
`import MyPkg` does not drag in GLFW (`[weakdeps]` + `[extensions]`, with the
call site resolving lazily via `Base.get_extension`). The `apt-get` line is
only a workaround for the *test* suite; the extension is the real fix.

---

## Test layout: flat `test/`, one aggregator

**Keep every test file at one level inside `test/` and aggregate them in
`test/runtests.jl`.** `Pkg.test` only ever picks up **`test/runtests.jl`** (a
missing one is the error `"Package X did not provide a 'test/runtests.jl'
file"`), so a nested `test/<subdir>/<file>.jl` is **never discovered**:
include it from `runtests.jl` and you pay for the nesting for nothing; don't
include it and it is dead code. Either way it is broken — nested files end up
**dead or run twice**. Prefer **named `@testset`s** around each `include`, so a
failing file is attributed to a named testset in the CI log instead of an
anonymous stack frame.

### The local-only gate flag

Define the flag in `test/runtests.jl` **before** the `include`s, so the
included files can see it. Two legitimate uses: **whole-file gating**
(`if RUN_LOCAL_ONLY; include("Local-Heavy.jl"); end`) and **per-assertion
gating** (`FLAG && @test ...`).

⇒ **The general trap: `FLAG && @test ...` makes a failure silently ignored
when `FLAG` is false.** The `@test` macro is never reached, so it does not
even appear as a *skip* in the summary — the assertion is simply invisible to
CI. In one observed file 135 of 186 `@test` calls were written that way, so
~73 % of that file never executed on CI. **Prefer `if FLAG ... end` around a
`@testset`**, so the tests show up as skips.

⇒ **A committed test file that no `include` references is dead weight that
silently rots.** Grep for orphans when you take over a repo.

---

## Intermediate output: `test/output/`

**Never write test artefacts into the source tree.** Use `test/output/` (or a
repo-root `output/`) and gitignore it with `test/output/`.

- The test creates the directory itself at run time with plain
  `mkpath(joinpath(@__DIR__, "output"))` — `mkpath` is Base and already
  idempotent, so **no helper is needed**.
- **Coverage artefacts are ignored by PATTERN, not by directory** —
  `*.jl.cov` and `*.jl.*.cov` (or plain `*.cov`). A directory rule for
  `output/` will not catch a `.cov` file written next to the source.
- ⇒ **A stale `*_report.json`, `*_progress` marker, or `stop` control file
  left in the output directory is an *unreviewable credential*.** It is not
  inert garbage: the next run can read it and conclude "the previous step
  passed" / "stop now" — a conclusion no reviewer can check. **Clear the
  directory at the start of each run, or write to a per-run subdirectory.**
- The danger is entirely **local**: CI clones fresh, so it always starts with
  an empty output dir; a developer's stale `test/output/` silently changes
  what the next local run does.

---

## Coverage: the badge

**Two badge URL shapes.** *Legacy*
`codecov.io/github/OWNER/REPO/graph/badge.svg?token=TOKEN` — do **not** copy
it: it embeds a **write-capable upload token** in a public README. If you find
one, treat that token as compromised and rotate it. *Modern, tokenless* —
the one to use: `https://codecov.io/gh/OWNER/REPO/graph/badge.svg`.

⇒ **The branch segment is optional but must be right.** `/branch/master/`
pins the badge; rename the default branch and the badge **does not error** — it
silently keeps rendering the last value computed on the old branch, so the
README advertises stale coverage behind a green badge. A **branch-less** URL
follows the default branch. **Audit the badge when you rename a default
branch.**

**Where the token lives:** `CODECOV_TOKEN` → **Settings → Secrets and variables
→ Repository → Actions → New repository secret**. Never commit it, never put
it in a badge URL. A **repository** secret is the right scope; an organisation
secret leaks to every repo in the organisation, including forks.

---

## Triage when CI turns red after a version bump

Run these *before* the bump, not after the red:

1. **Every direct dependency has a `[compat]` bound** — a missing bound
   resolves to "any version" and breaks in someone else's environment.
2. **The CI matrix covers the Julia version you are claiming to support** —
   otherwise the claim is untested. Compare the matrix against
   `[compat] julia` *and* the README.
3. **The coverage badge's pinned branch is still the default branch** —
   `git symbolic-ref refs/remotes/origin/HEAD`.
4. **No absolute paths in tracked files** — a `[sources]` entry written by
   `Pkg.develop` breaks every non-Windows machine (`SKILL.md` §6.4).

If CI is red after the bump, triage in this order — the first three find
resolution/environment causes rather than your own bug:

1. **Is the red real or a soft failure?** Read the *step* conclusions
   (`continue-on-error` can leave the job green).
2. **Was it a `pull_request` run?** Then the commit markers never applied.
3. **Is it resolution, not code?** The first error decides:
   `Unsatisfiable requirements detected for package X`,
   `expected package X [uuid] to exist at path ...`, or
   `Package X not found in current path` ⇒ fix `Project.toml` `[compat]`, not
   `src/`. A dependency's 1.0 is always a major bump for you.
4. **Re-resolve from scratch** — `Manifest.toml` is gitignored, so your local
   one is a stale artefact, not a record of what CI tested. Delete it,
   re-resolve, and inspect the resulting manifest for the *exact* sibling
   versions the test ran against.
5. **Is it the GLFW/X-server class?** Signature: a GLFW error at
   `using GLMakie` with **zero** test results printed.
6. **Did the docs break in the same commit?** A `workflow_dispatch`-only doc
   workflow makes a docs break invisible in CI — check `docs/make.jl` yourself.
7. **Only now, look at the code.**

---
name: julia-package-dev
description: "Ship and maintain a Julia package: version bumps and release, [compat] maintenance, keeping the docs building, CI and coverage, General-registry registration, and the local dev loop. Use when the user wants to release/publish/version-bump a Julia package, register one, fix a failing docs build or a GitHub Pages 404, update dependency compat bounds, or set up CI. Trigger on: Julia package, release, publish, version bump, register, General registry, TagBot, Registrator, CompatHelper, Documenter, makedocs, deploydocs, Pages 404, doc build failure, coverage badge, [compat], Julia CI."
---

# Julia package development

A **playbook of the things you cannot look up** — the traps, the misleading
symptoms, the ordering constraints, and the house conventions.

Anything the official documentation already states is **linked, not
restated**: [Pkg](https://pkgdocs.julialang.org/dev/),
[Documenter](https://documenter.juliadocs.org/stable/),
[GitHub Actions](https://docs.github.com/en/actions),
[GitHub Pages](https://docs.github.com/en/pages),
[CompatHelper](https://github.com/JuliaRegistries/CompatHelper.jl),
[Registrator](https://github.com/JuliaRegistries/Registrator.jl).

Everything here is general to any Julia package. Where a specific error
string is quoted, it is quoted verbatim so you can grep for it.

| Need | Where |
|---|---|
| Order of operations for a release | [§1](#1-release) |
| Dependency bounds | [§2](#2-compat) |
| Docs build fails / site 404s | [§3](#3-docs) + `references/` |
| CI, test layout, coverage | [§4](#4-ci) + `references/ci-and-coverage.md` |
| General registry | [§5](#5-registration) |
| House conventions | [§6](#6-conventions) |
| Symptom → first check | [§7](#7-first-check) |

---

## 1. Release

**The order is the content.** Steps 1–4 are where releases actually break.

1. **Bump `version` in `Project.toml`.** The registry reads it *from the
   tagged commit*, not from your working tree — a version bump that is not
   in the tagged commit is a release that does not exist. The `uuid` is
   permanent; never change it.
2. **Update `[compat]`** ([§2](#2-compat)) and write the release notes.
3. **Re-resolve from a clean environment and re-test.** A stale
   `Manifest.toml` in your tree can hide a real break.
   ```powershell
   Copy-Item Manifest.toml Manifest.toml.bak -ErrorAction SilentlyContinue
   Remove-Item Manifest.toml -ErrorAction SilentlyContinue
   julia --project=. -e 'using Pkg; Pkg.instantiate(); Pkg.test()'
   ```
4. **Check the docs build** and that `deploydocs` is actually *called*
   (a commented-out `deploydocs` gives a green CI and a permanent 404).
5. **Check the CI matrix covers the Julia version you are claiming to
   support.** Otherwise the claim is untested.
6. **Tag the commit that contains the version bump.**
   ```powershell
   git commit -am "release v0.2.2"
   git tag -a v0.2.2 -m "v0.2.2"
   git push origin main; git push origin v0.2.2
   ```
   If the repo uses **TagBot** ([§5](#5-registration)), do **not** push the
   tag yourself — push the commit and let TagBot tag it. Doing both is a
   conflicting-tag failure.
7. **Verify a stranger can install it** — the only real acceptance test:
   ```julia
   julia --project=@temp -e 'using Pkg; Pkg.add(name="MyPkg", version="0.2.2"); using MyPkg'
   ```
   It must resolve from the registry, not from a local path.

### Breaking changes must be declared

**Below 1.0 the *minor* version is the breaking axis** — `0.1 → 0.2` is the breaking release. So:

- **Any bump of the breaking axis (`0.x → 0.(x+1)`, or `1.x → 2.0`) MUST
  ship a release note containing a `## Breaking Changes` section** — even if
  it says "none". The section is the signal that the bump was deliberate;
  without it a reader cannot tell a considered break from an accidental one.
- **Never bump the breaking axis without breaking anything** — that burns the
  one major version your users get for free to fix a typo.
- A **patch** bump (`0.1.2 → 0.1.3`) must not contain breaking changes; if it
  does, it was a breaking-axis bump.
- A breaking change that slips into a patch release is the worst outcome: users
  get it with no version signal, so their `[compat]` bounds cannot protect them.

**Bumps people forget:** the `docs/` environment has its own `[compat]`; the
CI Julia matrix; the `[compat]` bounds *in sibling packages* that depend on
yours; and the coverage badge's pinned branch ([§4.4](#44-coverage)).

**Versioning.** The decision is "what breaks for a user", not "what did I
change": adding a keyword argument with a default is minor; renaming or
removing an export is **breaking** even if it is one function; changing the
*meaning* of an existing argument is breaking even though the signature is
unchanged — a version number cannot catch that, so review it by hand.

---

## 2. Compat

`[compat]` is what makes resolution reproducible for your users. Two
non-obvious things dominate the failures:

**The caret means different things at `0.x` and `1.x`.** `SomePkg = "0.1"`
is `>=0.1.0, <0.2.0` — it does **not** allow `0.2.0`. For `0.x` packages
the minor version is the breaking axis, so write the versions you actually
tested rather than a bare `"0.1"`:
```toml
GLMakie = "0.11, 0.12, 0.13"   # a tested set, not a range
CSV = "0.10, 1"                # 0.10 plus 1.x
```

**A 1.0 release of a dependency is always a major bump for you.** Real
cases seen in the wild:

| dependency | old bound | new release | what broke |
|---|---|---|---|
| `CSV` | `"0.9"` | `CSV 1.0` | resolution fails for users |
| `DataFrames` | `"0.22"` | `DataFrames 1.0` | `Base.length(::DataFrame)` removed → `MethodError` |
| `JLD2` | `"0.4"` | `JLD2 0.5` | API drift |

If you must support both majors, write both (`CSV = "0.10, 1"`) and
**actually test both** — an untested bound is a guess.

Every direct dependency needs a bound. A missing bound resolves to "any
version" and the break arrives in someone's environment, not yours.

**Keeping bounds current.** [CompatHelper](https://github.com/JuliaRegistries/CompatHelper.jl)
opens a PR when a dependency releases a version your bound excludes. Two
workflows, and the split matters:

- `CompatHelper.yml` — **scheduled** (cron), opens the PR.
- `CompatHelperMessage.yml` — runs only when the commit message contains
  the bracketed `[compat]` token, i.e. it applies the bound on merge.

The PR is *opened* automatically but the *merge* is what updates the bound —
keep the human in that loop. Real-world commits often use the bare word
(`compat: add Statistics [skip ci]`) and never the bracketed token, so the
apply workflow fires only when a human writes it by hand.

---

## 3. Docs

Two independent things can break, and they need different diagnosis:

- **the build fails** → `references/doc-deploy-traps.md`
- **the build is green but the site 404s** → `references/github-pages-and-deploy-auth.md`

### 3.1 The local build does not reproduce the CI failure

**`docs/Manifest.toml` is gitignored and is usually stale** — your local env
may still hold Documenter 0.26 while CI resolves 1.19, so the bug you are
chasing is invisible locally. **Re-resolve the way CI does before debugging
anything**, and remember that `Pkg.develop` then dirties the tracked
`docs/Project.toml` with a `[sources]` absolute path (revert it with
`git checkout HEAD -- docs/Project.toml`). Full recipe in
`references/doc-deploy-traps.md`.

### 3.2 Documenter 1.x changed defaults that break a 0.27-era manual

`checkdocs` now defaults to `:all` ⇒ every internal docstring is a hard error
(`checkdocs = :none` keeps the real check). `HTML(assets=…)` accepts only
`ico`/`css`/`js` ⇒ `Unrecognised asset class`. A module docstring with
`$(EXPORTS)` is itself checked ⇒ floods `:cross_references`. `strict` became
`warnonly`. Symptom table and fixes:
`references/doc-deploy-traps.md`.

### 3.3 "CI is green but the site 404s"

**A green workflow only proves the push to `gh-pages` worked.** If Pages was
never enabled for that repo, nothing serves it and the URL 404s forever.

**One field decides it:** does `GET /repos/{o}/{r}/environments` contain a
`github-pages` entry? Present ⇒ enabled ⇒ a transient 5xx ⇒ **re-run**. Absent
⇒ never enabled ⇒ a **manual** Settings step, and **re-running is useless**.
(`GET .../pages` 404s for working repos too — it is not a discriminator.)
Full diagnostic and decision flow: `references/github-pages-and-deploy-auth.md`.

### 3.4 Deploy authentication

Documenter branches on **whether `DOCUMENTER_KEY` is set** — unset means it
uses the auto-injected `GITHUB_TOKEN` over HTTPS, needing only
`permissions: contents: write` and **no SSH key at all**.

> ⚠ **The trap:** a leftover `DOCUMENTER_KEY` secret silently flips Documenter
> back to the SSH path, which fails with a misleading "Base64-encoded string
> of the SSH private key" message. When migrating off SSH keys, **delete the
> secret**, not just stop referencing it.

---

## 4. CI

### 4.1 The shape that works

```yaml
name: CI
on: [push, pull_request, workflow_dispatch]
jobs:
  test:
    runs-on: ${{ matrix.os }}
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    strategy:
      matrix:
        version: ['1.12.6']
        os: [ubuntu-latest]
        arch: [x64]
    steps:
      - uses: actions/checkout@v4
      - uses: julia-actions/setup-julia@v2
        with: { version: ${{ matrix.version }}, arch: ${{ matrix.arch }}}
      - run: sudo apt-get update && sudo apt-get install -y xorg-dev mesa-utils xvfb libgl1 freeglut3-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libxext-dev
      - name: Install Julia dependencies
        shell: julia --project=monorepo {0}
        run: |
          using Pkg;
          pkg"dev ."
      - name: Run the tests
        continue-on-error: true
        run: >
          DISPLAY=:0 xvfb-run -s '-screen 0 1024x768x24' julia --project=monorepo -e 'using Pkg; Pkg.test("MyPkg", coverage=true)'
      - uses: julia-actions/julia-uploadcodecov@v4
        env:
          CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

Four things in there are not obvious:

- **`continue-on-error: true` means a green job can hide failed tests.**
  Know which mode you are in before trusting a tick.
- **GLMakie needs an X server.** Without the `apt-get` line and `xvfb-run`,
  any test that plots dies with a GLFW error, not a plot error. A genuinely
  headless package should not need them at all — use `weakdeps` + a package
  extension so that `import MyPkg` does not drag in GLFW.
- **`--project=monorepo` + `pkg"dev ."`** is how a package is tested against
  its *sibling* packages in development. A plain `Pkg.test("MyPkg")` in the
  package env resolves the siblings from the **registry** — a different,
  often wrong, resolution. But `pkg"dev ."` only develops the package under
  test; dev the siblings explicitly if you need them local.
- **`github.event.head_commit` is null on `pull_request`** — so every
  commit-message gate above is **inert on PRs** and only fires on `push`.
  Do not rely on markers to control PR CI.

Details: `references/ci-and-coverage.md`.

### 4.2 Test layout — flat `test/`, one aggregator

**Put every test file at one level inside `test/` and aggregate them in
`test/runtests.jl`.** A nested `test/<subdir>/<file>.jl` is never discovered
by `Pkg.test` — it ends up dead or run twice.

```julia
using Test
using MyPkg

mkpath(joinpath(@__DIR__, "output"))

const RUN_LOCAL_ONLY = false

@time @testset "Unit" begin
    include("test_unit.jl")
end

if RUN_LOCAL_ONLY
    include("Local-Heavy.jl")
end
```

`mkpath` is Base and already idempotent — no helper needed. Define the gate
flag **before** the `include`s so the included files can see it, and gate with
`if … end`, never `FLAG && @test`.

> ⚠ **Trap, general to any package:** `FLAG && @test ...` makes a failure
> *silently ignored* when `FLAG` is false — the `@test` macro is never
> reached, so it does not even appear as a skip in the summary.

### 4.3 Intermediate output

**Never write test artefacts into the source tree.** Use `test/output/` (or a
repo-root `output/`) and gitignore it:

```gitignore
test/output/
docs/build/
docs/site/
temp/
```

A stale `*_report.json` or `*_progress` left in the tree is an
*unreviewable credential* — a later run can read it and conclude a step
passed. Clear the directory at the start of each run.

### 4.4 Coverage

Register at [codecov.io](https://codecov.io/), put `CODECOV_TOKEN` in
**Settings → Secrets and variables → Repository secrets**, and badge the
README:

```markdown
[![codecov](https://codecov.io/gh/OWNER/MyPkg.jl/graph/badge.svg)](https://codecov.io/gh/OWNER/MyPkg.jl)
```

**The branch segment is optional but must be right.** `/branch/master/`
pins the badge to a branch; rename the default branch and the badge
silently goes stale — it still renders, showing an old value. A
branch-less URL follows the default branch. A badge pinning `master` on a
`main` repo is already stale this way.

---

## 5. Registration

Official: [Pkg — Registries](https://pkgdocs.julialang.org/dev/registries/),
[General](https://github.com/JuliaRegistries/General),
[Registrator](https://github.com/JuliaRegistries/Registrator.jl),
[TagBot](https://github.com/JuliaRegistries/TagBot).

```
bump version → tag the commit → registry PR → human merges
```

**TagBot** watches for a `Register New Version` comment on a closed release
and creates the tag, which triggers the registry PR. Keep it restricted to
the bot account, or a random commenter can make you publish:

```yaml
on:
  issue_comment:
    types: [created]
  workflow_dispatch:
jobs:
  TagBot:
    if: github.event_name == 'workflow_dispatch' || github.actor == 'JuliaTagBot'
    steps:
      - uses: JuliaRegistries/TagBot@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

**New package** → [Registrator](https://github.com/JuliaRegistries/Registrator.jl):
```julia
using JuliaRegistrator
@JuliaRegistrator register
```
It opens a PR against General (a *compressed* registry — `General.tar.gz` +
`General.toml`, which is why a local checkout of `General` is usually
absent). A human merges it.

Before you register: the `uuid` is unique and permanent; the repository is
public; the package **loads from a clean environment** (the registry's CI
does `Pkg.add` + `using`, so a missing runtime dependency fails there, not
on your machine); every dependency has a `[compat]` bound.

**A rename means a new uuid and a new registry entry** — users of the old
name keep resolving to the old one. Keep the old name registered as a shim
that re-exports and warns.

---

## 6. Conventions

Working habits worth keeping. Not Julia rules — choices.

### 6.1 No submodules

A submodule is addressed by a **relative path**, so every reference is a
`..`-style expression to resolve — a permanent tax on the mental model, and
one that breaks the moment the tree moves (the path is written into a
config, a `Project.toml` or a build script). A name-based dependency has
neither problem.

**So: big enough to warrant a submodule → make it a package.** Small →
vendor the **files** (with licence and provenance in a header) or take a
normal dependency. Never reach for a submodule as a third option.

A new sibling package is nearly free — own `Project.toml` (own `uuid`),
own `test/runtests.jl`, own `docs/`, own README with a badge, own registry
entry — and the consumer depends on it by name and UUID.

### 6.2 At most two levels of folders

```
MyPkg/
├── src/            # the package (one or two subdirectories max)
├── test/           # all test files flat, see §4.2
├── docs/           # make.jl, Project.toml, src/
├── examples/
└── .github/workflows/
```

Deeper nesting in `src/` means the `include` graph grows and a rename
becomes a sweep. Test data belongs in `test/output/` (gitignored), not in a
`test/data/<format>/<variant>/` tree.

### 6.3 `Manifest.toml` is gitignored

```gitignore
Manifest.toml
```

A `Manifest.toml` pins an *exact* resolution; committing it to a
**library** forces every user and every CI run onto those versions,
including versions that do not exist for their platform. Commit one for
*applications*, not libraries.

> ⚠ Corollary: the `Manifest.toml` in your tree is a **local artefact with
> a shelf life**, not a record of what was tested. It goes stale silently.
> When a bug "cannot be reproduced locally", suspect it first
> ([§3.1](#31-the-local-build-does-not-reproduce-the-ci-failure)).

### 6.4 `[sources]` pollution

`Pkg.develop(path=...)` writes a **machine-specific absolute path** into
the tracked `Project.toml`:

```toml
[sources]
MyPkg = {path = "E:\\path\\to\\MyPkg"}
```

In git that breaks CI on every other machine. Revert before committing:
`git checkout HEAD -- docs/Project.toml`. `[sources]` is a Julia 1.11+
feature for "this dependency comes from a local path" — fine in a working
tree, never committed.

### 6.5 Local development: `dev` + Revise

```julia
using Pkg
Pkg.develop(path="E:/path/to/MyPkg")   # dev-install
using Revise                            # pick up source edits, no restart
```

- **Revise removes the "restart and wait 90 s" loop.**
- **Dev-install every sibling you are changing**, not just the one under
  test: `Pkg.develop([PackageSpec(path="../A"), PackageSpec(path="../B")])`
- **A dev-installed package is invisible to `--project=docs`** — the docs
  env is separate and needs its own `Pkg.develop`.

Prefer `Pkg.develop(path=...)` over `Pkg.develop(url=...)`: the path form
is instant and works offline.

### 6.6 Commit-message markers

Mark **every** commit, and let the workflows key on it:
`[skip ci]` (skip the matrix), `[compat]` (apply compat bumps),
`[doc]` (build the docs). Remember these only affect `push` — see
[§4.1](#41-the-shape-that-works).

---

## 7. First check

| Symptom | First thing to check |
|---|---|
| Docs build fails locally but passes in CI (or the reverse) | stale `docs/Manifest.toml` → [§3.1](#31-the-local-build-does-not-reproduce-the-ci-failure) |
| CI green, site 404s | `has_pages` / `environments` → [§3.3](#33-ci-is-green-but-the-site-404s) |
| `git@github.com: Permission denied (publickey)` | a leftover `DOCUMENTER_KEY` secret → [§3.4](#34-deploy-authentication) |
| `Unrecognised asset class` | Documenter 1.x `assets` restriction → `references/doc-deploy-traps.md` |
| `:missing_docs` after a Documenter bump | `checkdocs` default flipped to `:all` → [§3.2](#32-documenter-1x-changed-defaults-that-break-a-027-era-manual) |
| `Package X not found in current path` in a doc block | missing `[deps]` entry in `docs/Project.toml` → `references/doc-deploy-traps.md` |
| `[sources]` path in a diff | `Pkg.develop` side effect → [§6.4](#64-sources-pollution) |
| Users cannot install your new version | version in `Project.toml` ≠ the tag → [§1](#1-release) |
| A dependency's 1.0 broke you | compat written for the old major → [§2](#2-compat) |
| Codecov badge shows stale values | branch renamed, or badge URL shape wrong → [§4.4](#44-coverage) |
| Tests pass locally, fail in CI | `Pkg.test` resolved different sibling versions → [§4.1](#41-the-shape-that-works) |
| A commit marker did nothing | `head_commit` is null on PRs → [§4.1](#41-the-shape-that-works) |

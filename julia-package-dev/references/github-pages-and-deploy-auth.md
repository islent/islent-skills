# GitHub Pages deployment & deploy authentication (non-obvious only)

> The **publishing-side** traps: the misleading symptoms and the verified fixes
> that GitHub's and Documenter's own docs do not state.
> Every rule is **general to any Julia package**.
> Official material is **linked, not restated**: [GitHub Pages settings](https://docs.github.com/en/pages),
> [Documenter `deploydocs`](https://documenter.juliadocs.org/stable/man/deploydocs/),
> [`julia-actions/julia-docdeploy`](https://github.com/julia-actions/julia-docdeploy).
> Build-side traps are in [`doc-deploy-traps.md`](doc-deploy-traps.md) — this file
> assumes the build is green.

---

## "CI is green but the site 404s"

`deploydocs()` succeeding **only means the push to `gh-pages` worked.** Three
independent stages: **build** (`makedocs`) → the Julia exit code; **push**
(`deploydocs`) → a new commit on `gh-pages`; **publish** (the Pages service) →
the site actually serves content. A green workflow proves the first two. The
third is a separate, asynchronous, server-side step that produces **no job of
your own** — its truth lives in `deployments` / `environments`, never in your
exit code.

⇒ **"CI green" ≠ "site live."** Never accept a green run as evidence of publication.

### The diagnostic (anonymous, no token)

| Read | Never enabled | Enabled |
|---|---|---|
| `GET /repos/{o}/{r}` → `has_pages` | `false` | `true` |
| `GET /repos/{o}/{r}/environments` | **`[]`** | one `github-pages` entry |
| `GET /repos/{o}/{r}/deployments` | `[]` | records with `environment_url` = live URL |
| `GET /repos/{o}/{r}/pages` | 404 | 404 — **useless** |

**TRAP — `GET /repos/{o}/{r}/pages` has zero discriminating power.** It 404s for
*every* repo, including ones whose site is demonstrably live. The endpoint
requires authentication, so an anonymous 404 is an **auth artifact, not a
state**. The tempting conclusion "`/pages` 404 ⇒ Pages was never enabled" is
**wrong**.

**TRAP — `has_pages: false` means "this repository has no Pages site", not
"Pages is configured with the wrong source".** Two different problems, identical
in the UI.

### The two faults that look identical

Both present as "CI green, page 404 / stale". They require **opposite** actions.

| | (a) Pages never enabled | (b) Pages server-side transient 5xx |
|---|---|---|
| `environments` | **no `github-pages` entry** | has `github-pages` |
| `deployments` | `[]` | latest `state` is `deployment_failed` |
| Error text | none (your workflow never fails) | `Deployment failed, try again later` |
| Fix | manual Settings change | **Re-run the workflow** |
| Re-run useful? | **No — completely useless** | **Yes — this is the fix** |

**THE DISCRIMINATOR is one field: does `environments` contain a `github-pages`
entry?** Present ⇒ enabled ⇒ read `deployments[].statuses[].state` and re-run if
it is `deployment_failed`. Absent ⇒ never enabled ⇒ only the manual step helps;
re-running pushes `gh-pages` again and changes nothing.

**(a) The manual step.** Settings → Pages → Build and deployment → Source:
"Deploy from a branch" → Branch: `gh-pages` → Folder: `/(root)` → Save. There
is no usable API equivalent without an admin token ⇒ **this is a human-only
step**; hand over one click sequence rather than working around it. It makes
Pages serve the branch directly, so publishing no longer depends on the
`pages-build-deployment` workflow or the artifact upload. Observed immediately
after Save: GitHub creates the `github-pages` **environment**, auto-creates the
dynamic workflow, runs it once, and produces a `deployments` record with the live
URL — and that run's `head_sha` is the **already-pushed** gh-pages commit ⇒
**enabling Pages publishes the existing content without a new Documenter push.**

**(b) The transient 5xx.** `Deployment failed, try again later` is a **generic
placeholder** — what the deploy action emits whenever the Pages API answers
`status='deployment_failed'` (server 500, rate limit, metadata desync, OIDC
state). The service deliberately does not return the underlying code.
⇒ **Re-run the workflow, change nothing.** A "success → failure → re-run
success" pattern is the signature of a transient server error; a re-run deploys
the *latest* commit, not the failed one.

### Proof the push itself succeeded

The gh-pages HEAD commit is the decisive artifact: **author**
`Documenter.jl <documenter@juliadocs.github.io>`, **subject**
`build based on <source-sha>`, timestamp just before the job finished. The
author and message format are produced by `deploydocs` itself, so their
presence proves *Documenter* pushed — not a human, not CI's own git step.
**`Documenter.jl` as the gh-pages author is normal and is not an SSH-key
symptom.**

**TRAP — Documenter 1.x writes `versions.js`, not `versions.json`.** Grepping
the tree for `versions.json` to confirm a deploy gives a false negative. Same
for `.nojekyll`: absent from both the broken and the working repo, both served
correctly ⇒ not the fault; do not chase it.

---

## Deploy authentication: GITHUB_TOKEN vs SSH deploy key

Documenter's own source, `src/deployconfig.jl`:

```julia
authentication_method(::GitHubActions) = env_nonempty("DOCUMENTER_KEY") ? SSH : HTTPS
```

⇒ **The presence of `DOCUMENTER_KEY` is the only switch.** Unset ⇒ `deploydocs`
uses HTTPS with the auto-injected `GITHUB_TOKEN` and needs **no SSH key at
all** — only `permissions: contents: write` on the workflow. (The default
`GITHUB_TOKEN` is read-only; without that permission `deploydocs` fails
**silently**: no error, CI green, `gh-pages` not updated.)

The asymmetry that causes this whole class of bugs: the deploy *criteria* check
is satisfied by `GITHUB_TOKEN` **or** `DOCUMENTER_KEY`, but the *transport* is
chosen by `DOCUMENTER_KEY` alone. A leftover key therefore passes the criteria,
selects SSH, and fails at the git layer with a message about the key.

### The SSH failure signatures (verbatim)

```
git@github.com: Permission denied (publickey)
fatal: Could not read from remote repository.
```

Documenter's wrapper adds:

```
┌ Error: Git failed to fetch git@github.com:ORG/Pkg.jl.git
│ This can be caused by a DOCUMENTER_KEY variable that is not correctly set up.
│ Make sure that the environment variable is properly set up as a Base64-encoded
│ string of the SSH private key.
```

**TRAP — the message says "Base64-encoded string of the SSH private key", but
the real problem usually is not Base64.** In the observed case it was a stale /
revoked / unregistered key. Documenter names the *format* because that is the
only thing it can check locally; it cannot know the key was revoked
server-side.

### THE TRAP when migrating off SSH keys

**A leftover `DOCUMENTER_KEY` flips Documenter back to the SSH path.** The
dangerous case is a live `env: DOCUMENTER_KEY` line in a workflow that also
exports `GITHUB_TOKEN`: the two coexist, SSH wins, and the failure message
talks about Base64 SSH keys — sending you down exactly the wrong rabbit hole. A
key that is *referenced but unused* (still in `env:` while `deploydocs` was
commented out) is **not inert**: the moment `deploydocs()` is re-enabled it
silently switches the transport back.

⇒ **Delete the secret, not just stop referencing it.** A secret left in the
repo's secrets list keeps the footgun loaded for whoever re-adds the
reference.

⇒ **Audit every workflow whenever you (re-)enable deployment**, not just the
one you are editing: `Select-String -Path .github/workflows/*.yml -Pattern 'DOCUMENTER_KEY'`.
A commented-out `# ssh: ${{ secrets.DOCUMENTER_KEY }}` line is exactly how
these get re-enabled by accident later.

⇒ **Stale comments in workflow files are not evidence.** A workflow comment
claiming "deploydocs is disabled" was wrong — `docs/make.jl` had it enabled.
Read `docs/make.jl` itself.

**Migration order:** delete the `env: DOCUMENTER_KEY` reference *and the
secret*; add `permissions: contents: write`; **keep the `deploydocs()` call in
`docs/make.jl`** (deleting it because you switched to
`julia-actions/julia-docdeploy@v1` is the single most common mistake); bump
`actions/checkout@v1` → `@v4` and `setup-julia@latest` → `@v2`; pin
`DOCUMENTER_VERSION` explicitly with the action.

**Two auth paths can coexist across a group of repositories** — do not
"unify" blindly. A hand-written `run: julia --project=docs docs/make.jl` step
and a `uses: julia-actions/julia-docdeploy@v1` step both appear in the wild,
and the hand-written one exists for a reason (a package pinning
`compat Documenter = "1"` conflicts with the action's pinned
`DOCUMENTER_VERSION`). **Check what each repo actually has.**

---

## deploydocs vs julia-actions/julia-docdeploy

`julia-actions/julia-docdeploy@v1` is a **wrapper**: it `Pkg.develop`s the
package into the docs environment, `Pkg.instantiate`s, and then
**`include("docs/make.jl")`** — which calls both `makedocs` and `deploydocs`.

⇒ **`docs/make.jl` must still call `deploydocs()` itself.** The action does
not deploy; it delegates the whole deployment decision back to your `make.jl`.

**TRAP — deleting `deploydocs()` from `make.jl`** on the assumption that "the
Action deploys automatically" gives `conclusion: success`, no site update and
no new `/deployments` record. The action ran `makedocs`, then exited. This is
**identical to "the push failed"** from the outside. Discriminate by checking
the gh-pages HEAD: if it did not advance, `deploydocs()` never ran.

**TRAP — the action's `prefix:` input is word-split by the shell.** It is
embedded as a single bash array element, so `prefix: xvfb-run -a -s "-screen 0
1024x768x24"` delivers the quoted literal to `xvfb-run` and Xvfb rejects it
with `0: command not found` / exit code 127. ⇒ `prefix:` is only usable for a
**no-argument** wrapper; anything more complex must move to a hand-written step.

### The `deploydocs` kwargs that matter

- **`repo`** — required, no default. The criteria check requires
  `GITHUB_REPOSITORY` (e.g. `ORG/Pkg.jl`) to **occur in** this string, so
  `repo = "github.com/ORG/Pkg.jl.git"` passes.
- **`target`** — the built directory, **relative to the docs root**; it must be
  a **subdirectory of `root`**, else Documenter errors with
  `target must be a subdirectory of root, got: ...`.
- **`devbranch`** — matters whenever the default branch is `main` and not
  `master`: Documenter's default comes from a `git remote show origin` probe
  with a **`"master"` fallback**, so a wrong value silently deploys nothing.
  Documenter 1.x warns `Possible deploydocs() misconfiguration: main vs master`.
  `devbranch = nothing` auto-detects, but **set it explicitly** — copying a
  `docs/make.jl` between repos requires checking each repo's default branch
  first.

### The first-deploy path

A repo that has never deployed has no `gh-pages` branch. `deploydocs` tries
`git checkout -b gh-pages upstream/gh-pages` and falls back to
`git checkout --orphan gh-pages` + an empty commit. ⇒ **The very first run can
succeed**, and you will see exactly two commits: an empty "Initial empty
commit for docs" then "build based on <sha>". Do not read the orphan commit as
a bug. (Pages may still need the manual enablement afterwards.)

### `deploydocs` commented out in `make.jl`

⇒ green CI, no `gh-pages` branch, 404 forever, and any `DOCUMENTER_KEY` in the
workflow is dead configuration. **Check that `deploydocs` is actually called**
(`Select-String -Path docs/make.jl -Pattern 'deploydocs' -Context 0,4`).

Build-side failures (`checkdocs` / `warnonly` / a stale `docs/Manifest.toml`)
masquerade as deploy failures — a `makedocs` stage that errors never reaches
`deploydocs`, so `gh-pages` does not advance and the symptom is "the site
still shows the old version". Confirm the Julia process exited 0 before
touching anything deployment-related.

---

## Evidence-collection recipes

All anonymous (until the 60 req/h per-IP rate limit); `gh` is often not
installed, so use `Invoke-RestMethod` / `curl`.

| Question | Where |
|---|---|
| Does the repo have a Pages site? | `GET /repos/{o}/{r}` → `has_pages`, `homepage` |
| **Was Pages ever enabled?** | `GET /repos/{o}/{r}/environments` → is there a `github-pages` entry? |
| What did the last publish do? | `GET /repos/{o}/{r}/deployments` then `.../deployments/<id>/statuses` → `state`, `environment_url` |
| Did Documenter's push land? | `git show -s --format="%an <%ae>%n%s%n%cI" origin/gh-pages` → author + subject |
| Is the site actually serving? | `GET https://ORG.github.io/Pkg.jl/dev/` |

### TRAP — the same URL can serve a STALE cached body

During one investigation the *identical* request
`https://api.github.com/repos/ORG/Pkg.jl` returned `has_pages: false` with an
old `updated_at`, while the same path with a cache-busting query (`?cb=1`)
returned `has_pages: true` with a current one — and repeating the *plain* URL
returned the stale body again, deterministically.

⇒ **A stale cached `has_pages: false` will make you re-diagnose an
already-fixed repo, or conclude the fix failed.** Three defences: **1.** compare
`updated_at` against the time of the event you are investigating — if it
predates your deploy, the body is stale; **2.** cache-bust the URL whenever the
conclusion would flip the diagnosis; **3.** require agreement between
`has_pages`, `environments`, `deployments` and `actions/workflows` before
concluding anything, and prefer `environments` (the discriminator).

---

## Consistency across a family of repositories is the goal

When you fix one repo, make the siblings match, so the next agent has one
pattern to learn rather than nine. The cheapest diagnostic shortcut: **when one
package's site 404s and its siblings' work, diff the *repo settings*, not the
code** — the code is nearly identical across a family of packages, and the only
difference that mattered was a **repo setting** no amount of code review could
reveal. Two settings carry that pattern: the **Pages source** (uniform
"Deploy from a branch / `gh-pages` / `/(root)`" for every repo that has a site)
and **`devbranch`** (must match each repo's real default branch).

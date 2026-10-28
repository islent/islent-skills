# islent-skills

Skills for LLM, maintained by islent.

Each skill is a self-contained directory with a `SKILL.md` (the entry point
and the playbook) and optional `references/` (the deep material it points
to). Skills are written for **agents**, not humans: they are task-oriented,
dense, and every claim is either verifiable in a repository or a link to the
official documentation.

## Available skills

| Skill | What it is for |
|---|---|
| [`julia-package-dev/`](julia-package-dev/) | Developing, versioning and releasing a Julia package: version bumps and release checklists, `[compat]` maintenance, keeping the docs building, CI and coverage badges, registering with the General registry, and the local development loop. Written for any Julia package. |

## Conventions

- **Official documentation is linked, not restated.** A reference file that
  only repeats what `Documenter`'s manual already says is noise; the value
  is in what the manual does not say — the traps, the misleading error
  messages, the false leads, and the bottlenecks that actually cost time.
- **Traps are written as symptom → cause → fix**, because that is the order
  you need them in when something breaks at 2 a.m.
- **Gaps between jobs are explicit.** If a step is a manual click in a web
  UI, it says so, because an agent cannot perform it and must tell the user
  instead of silently working around it.


---
name: setup-project
description: >
  Prepares a repository to work with the dev-pipeline plugin at project start:
  writes the `## Pipeline contract` section of CLAUDE.md, the docs layout
  (tasks, plans, ADRs, architecture), the first ADR, and checks branches and
  shadowing agents. Use when the user says "/setup-project", "настрой
  репозиторий под пайплайн", "подготовь проект под плагин", "старт проекта",
  or a dev-pipeline agent stopped with "CLAUDE.md has no Pipeline contract".
  Args (optional): short project description or stack hints.
---

Set up the current repository for the `dev-pipeline` plugin. The plugin's
pipeline is primary; everything specific to this project goes into the
project's `CLAUDE.md` (`## Pipeline contract`) and `docs/`. Talk to the user in
their language; write the files in English unless the user asks otherwise.

Never overwrite or delete the user's content: merge into existing files, and
show what you will change before writing anything that already exists.

## 1. Inspect (read-only)

- Repo state: `git rev-parse --show-toplevel` (if not a git repo, offer
  `git init` and stop until the user agrees), `git status`, current branch,
  local and remote branches (`git branch -a`), remotes (`git remote -v`), the
  forge (`gh repo view` / `glab repo view` if available).
- Stack and tooling, to infer gate commands: `pyproject.toml` (uv/poetry/pip;
  ruff, mypy, pytest), `package.json` (scripts: lint, format, typecheck, test;
  pnpm/yarn/npm by lockfile), `go.mod`, `Cargo.toml`, `Makefile`/`justfile`
  targets, CI workflows (`.github/workflows/*.yml`, `.gitlab-ci.yml`) — the CI
  commands are the best source for the gate.
- Test locations (`tests/`, `test/`, `**/*_test.go`, `**/*.test.ts`, ...).
- Existing `CLAUDE.md` (and `AGENTS.md`), `docs/` layout, existing ADRs or
  task files and their numbering.
- Project-level `.claude/agents/{manager,coder,tester,code-reviewer,qa-mr}.md`
  and `.claude/skills/run-task/`: same-named project agents shadow the plugin's.
- Dev servers, reserved ports, secret files (`.env`, `.env.example`) — note
  their existence only; never print their values.

## 2. Propose the contract

Fill every field of the template below from what you found, mark guesses, and
ask the user to confirm or correct in one message (use AskUserQuestion for the
real choices: branch model, forge, merge policy). Defaults when nothing says
otherwise:

- Branch model: `main` only (base and PR target `main`). Offer the
  `develop` + `main` release model as an option; if chosen, `develop` is the
  base and PR target and `main` changes only by a release PR on request.
- Forge: `gh` if the remote is GitHub and `gh` works, `glab` for GitLab,
  otherwise `none`.
- Merge/release policy: PR opened without asking; merge, direct pushes to
  protected branches, release PRs, tags and force-push only on the owner's
  explicit request; never propose merging a PR whose CI is red or pending.
- Docs layout: `docs/tasks/`, `docs/plan/`, `docs/adr/`, living
  `docs/arch/overview.md`.

```markdown
## Pipeline contract

- **Base branch:** <branch>
- **PR target:** <branch>
- **Forge:** <gh | glab | none>
- **Branch naming:** task/NNNN-slug; docs-only work on docs/<slug>
- **Docs layout:** tasks docs/tasks/, plans docs/plan/ (shared NNNN
  numbering), ADRs docs/adr/ (immutable), living docs: docs/arch/overview.md.
  Docs language: <English>.
- **Test paths:** <dirs or globs>
- **Gate commands:**
  - lint: `<cmd> <touched files>`
  - format: `<cmd> <touched files>`
  - types: `<cmd> <touched files>`        <!-- omit if none -->
  - tests: `<full test command>`
- **Security-shaped paths:** <app entrypoint, settings, auth/token/credential/
  secret/password/session paths, migrations/, ...>
- **Merge/release policy:** <as agreed>
- **Extra rules:** <reserved ports, processes not to touch, secrets handling,
  real-browser checks for front-end scripts, domain rules>
```

Verify each gate command actually runs before writing it in (run it once; if
it fails for reasons unrelated to the setup, say so and keep it, marked).

## 3. Write (after the user confirms)

1. `CLAUDE.md`: create it, or add/replace only the `## Pipeline contract`
   section, plus a short `## Agent pipeline` section:
   "Non-trivial work goes through the `dev-pipeline` plugin
   (`/dev-pipeline:run-task`); project specifics are the Pipeline contract.
   Do not add same-named `.claude/agents/` copies unless a deliberate
   override is needed." Keep the user's other content untouched.
2. Docs skeleton (only what is missing): `docs/README.md` (one paragraph per
   folder: tasks = what/why, plan = approach written before implementing,
   adr = immutable decisions, arch = current shape kept up to date),
   `docs/tasks/`, `docs/plan/`, `docs/adr/`, `docs/arch/overview.md` (a short
   current-shape stub from what you inspected — no invented facts).
3. First ADR (next free number, `0001` in a fresh repo):
   `docs/adr/NNNN-dev-pipeline-plugin.md` — Status: accepted; Context; Decision
   (pipeline from the `dev-pipeline` plugin, project specifics in the
   contract, branch model and merge policy as agreed); Consequences (the plugin
   must be installed on the machine; sessions load agents at start).
4. Shadowing agents found in step 1: list them and ask whether to remove them
   (the plugin's version is used then) or keep them as deliberate overrides.
   Never delete without a yes.
5. Branch model `develop`: if `develop` doesn't exist, offer to create it from
   the current `main` locally; pushing it is a separate yes.

## 4. Commit and publish

- Work on a branch `docs/setup-dev-pipeline` from the contract's base branch
  (or on the initial branch in an empty repo, with the user's consent).
- Commit only with an explicit pathspec (`git add -- <new files>`,
  `git commit -m "..." -- <paths>`); never `git add -A/./-u` — the index may
  hold the user's own staged changes.
- Push and open the PR into the PR target only after the user says yes (this
  is the first run, there is no merge policy in force yet). Forge `none`:
  leave the branch local.

## 5. Report

What was found, the contract as written, files created/changed, anything
skipped or left for the user (shadowing agents, `develop` push, failing gate
commands), and the next step: restart open sessions in this repo (or
`/reload-plugins`) and start work with `/dev-pipeline:run-task <task>`.

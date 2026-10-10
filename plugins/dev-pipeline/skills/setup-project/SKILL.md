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
- Stack and tooling: find every manifest from the stack table below, at the
  root and in subdirectories (skip `node_modules`, `.venv`, `vendor`, `target`,
  `build`, `dist`). Gate commands, in order of trust:
  1. what CI already runs (`.github/workflows/*.yml`, `.gitlab-ci.yml`, other
     CI configs) — copy those commands;
  2. what the project declares: `package.json` scripts, `Makefile`/`justfile`
     targets, `[tool.*]` sections and dev dependencies in `pyproject.toml`,
     configured linters (`.eslintrc*`, `ruff.toml`, `.golangci.yml`, ...);
  3. only then the stack table defaults, marked as guesses.
- Test locations (`tests/`, `test/`, `**/*_test.go`, `**/*.test.ts`, ...).
- Then classify the repo: **empty** (no manifest and no source files, see
  step 1a), **single stack** (one manifest set at the root), or **monorepo**
  (manifests in several subdirectories, workspaces such as
  `pnpm-workspace.yaml`, `package.json` `workspaces`, Cargo/Go/uv workspaces,
  or mixed stacks, see step 1b).

### Stack table (defaults when the project declares nothing)

| Marker | Stack | lint | format check | types | tests | test paths |
|---|---|---|---|---|---|---|
| `pyproject.toml` + `uv.lock` | Python (uv) | `uv run ruff check` | `uv run ruff format --check` | `uv run mypy` | `uv run pytest` | `tests/` |
| `pyproject.toml` + `poetry.lock` | Python (Poetry) | `poetry run ruff check` | `poetry run ruff format --check` | `poetry run mypy` | `poetry run pytest` | `tests/` |
| `requirements*.txt` / `setup.py` | Python (pip) | `ruff check` | `ruff format --check` | `mypy` | `pytest` | `tests/` |
| `package.json` + `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` / `bun.lockb` | JS/TS (pnpm / yarn / npm / bun) | `<pm> run lint` | `<pm> run format:check` or `prettier --check .` | `tsc --noEmit` (if `tsconfig.json`) | `<pm> test` | `**/*.test.{ts,tsx,js}`, `**/*.spec.*`, `tests/` |
| `go.mod` | Go | `golangci-lint run` (else `go vet ./...`) | `test -z "$(gofmt -l .)"` | (compiler) `go build ./...` | `go test ./...` | `**/*_test.go` |
| `Cargo.toml` | Rust | `cargo clippy --all-targets -- -D warnings` | `cargo fmt --check` | (compiler) | `cargo test` | `tests/`, inline `#[cfg(test)]` |
| `build.gradle(.kts)` / `pom.xml` | Java/Kotlin (Gradle / Maven) | `./gradlew check` / `mvn -q verify` (checkstyle, detekt, ktlint if configured) | spotless/ktlint check if configured | (compiler) | `./gradlew test` / `mvn test` | `src/test/` |
| `*.sln` / `*.csproj` | .NET | `dotnet build -warnaserror` | `dotnet format --verify-no-changes` | (compiler) | `dotnet test` | `*Tests/` projects |
| `Gemfile` | Ruby | `bundle exec rubocop` | (rubocop) | `bundle exec srb tc` if Sorbet | `bundle exec rspec` / `bin/rails test` | `spec/`, `test/` |
| `composer.json` | PHP | `vendor/bin/phpstan analyse` / `vendor/bin/psalm` | `vendor/bin/php-cs-fixer fix --dry-run` | (phpstan) | `vendor/bin/phpunit` / `vendor/bin/pest` | `tests/` |

A tool from the table that the project does not have is not a gate command:
offer to add it (step 1a list) instead of writing a command that cannot run.
Commands that take files (`<touched files>`) get the placeholder in the
contract; whole-project commands (compilers, `cargo test`) don't.

### 1a. Empty repo

Nothing to infer: ask a short survey (AskUserQuestion, at most 4 questions in
one call): language/stack; package manager; test framework; linter/formatter.
Put the recommended set first, from the table: uv + ruff + mypy + pytest;
pnpm + TypeScript + ESLint + Prettier + Vitest; Go modules + golangci-lint +
`go test`; cargo + clippy + rustfmt; Gradle + JUnit 5 (+ ktlint); dotnet +
xUnit; Bundler + RuboCop + RSpec; Composer + PHPStan + PHPUnit. Write the
contract from the answers, marking that the tools are not installed yet, and
offer (separately, as its own yes) to scaffold the minimal project files
(manifest, lint/format config, one passing test, `.gitignore`) so that every
gate command runs green before the first task. Don't install global tools.

### 1b. Monorepo

List the parts (directory → stack). Gate commands and test paths are given
per part in the contract (format in step 2); a root-level command that already
covers everything (e.g. `pnpm -r lint`, `turbo run test`, `nx affected`,
`make check`) is preferred over per-part ones when CI uses it. Each part's
commands state the working directory. Ask the user to confirm the part list.
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

Monorepo form of the two per-part fields (other fields stay shared):

```markdown
- **Test paths:** backend/tests/, frontend/**/*.test.ts
- **Gate commands** (per part; run a part's commands when the diff touches
  files under its directory, from that directory; `<touched files>` are
  relative to it):
  - `backend/` (Python, uv):
    - lint: `uv run ruff check <touched files>`
    - types: `uv run mypy <touched files>`
    - tests: `uv run pytest`
  - `frontend/` (TS, pnpm):
    - lint: `pnpm lint`
    - types: `pnpm exec tsc --noEmit`
    - tests: `pnpm test`
  - root (always): `<root command, if any>`
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

# dev-pipeline plugin (draft)

One subagent pipeline for every project: `manager` → `coder` →
`tester` ∥ `code-reviewer` ∥ `ponytail-review` → `qa-mr`, plus the `run-task`
skill that drives it.
The agents hold only the generic workflow; everything project-specific comes
from a `## Pipeline contract` section in each project's `CLAUDE.md`.

```
dev-pipeline-plugin/                 # a local marketplace ("gronck-local")
├── .claude-plugin/marketplace.json
└── plugins/dev-pipeline/
    ├── .claude-plugin/plugin.json
    ├── agents/{manager,coder,tester,code-reviewer,qa-mr}.md
    └── skills/{run-task,setup-project}/SKILL.md
```

## Review rules

- `code-reviewer` writes caveman-style (terse one-liners
  `<file>:L<line>: <severity>: <problem>. <fix>.`) and ends with `REVIEW_CLEAN`
  or `REVIEW_FINDINGS: <n>`.
- Findings are mandatory to fix. Rounds 1 and 2: the main thread sends every
  finding back to `coder` without asking. Round 3: `🔵 nit` findings are always
  skipped; every other finding is put to the user (fix / follow-up / accept).
- Every `code-reviewer` pass is the next round, whatever triggered it. Review
  rounds count toward the cap of 3 retry cycles per task.

## ponytail (optional)

With the [ponytail](https://github.com/DietrichGebert/ponytail) plugin
installed (`ponytail@ponytail`), the pipeline also checks for code that
shouldn't exist:

- `coder` and `tester` follow ponytail's smallest-complete-change rules. The
  key rules are copied into the agents, so they apply without the plugin too.
- `run-task` invokes the `ponytail-review` skill itself (inline, findings
  only) on the same diff as `code-reviewer`, while `tester` and
  `code-reviewer` run. Its **Must fix** / **Should fix** findings go into the
  same `coder` fix cycle and round as `code-reviewer`'s; **Nice to have**
  findings never block and are listed in the final report.
- `qa-mr` gets the ponytail-review result and fails the gate if a Must/Should
  finding is still open (not fixed and not accepted by the user).
- Without the plugin the step is skipped with `ponytail-review: skipped (not
  installed)` in the report; nothing else changes.

## Start a new project: `/dev-pipeline:setup-project`

Run it in the repo. It inspects the stack, CI and branches, proposes the
`## Pipeline contract` (gate commands checked by running them), and after your
confirmation writes it into `CLAUDE.md`, adds the docs skeleton
(`docs/tasks`, `docs/plan`, `docs/adr`, `docs/arch/overview.md`) and the first
ADR, flags same-named `.claude/agents/` copies that would shadow the plugin, and
offers a `develop` branch. It pushes or opens a PR only after you say yes.

Stack detection: CI commands first, then what the project declares (scripts,
Makefile targets, tool configs), then a built-in stack table (Python uv/Poetry/
pip, JS/TS pnpm/yarn/npm/bun, Go, Rust, Java/Kotlin, .NET, Ruby, PHP) as marked
guesses; every gate command is run once before it is written. An empty repo
gets a short survey with a recommended toolset (and optional scaffolding); a
monorepo gets per-part gate commands (directory → commands), which `coder`,
`tester` and `qa-mr` run only for the parts a diff touches.

## Install (user scope = every project)

The scratchpad is temporary. Move the directory somewhere permanent first,
because a marketplace added from a local path is loaded in place:

```bash
mv <scratchpad>/dev-pipeline-plugin ~/claude-plugins
claude plugin validate ~/claude-plugins
claude plugin marketplace add ~/claude-plugins
claude plugin install dev-pipeline@gronck-local          # --scope user is the default
claude plugin list                                       # dev-pipeline@gronck-local  ✔ enabled
```

`install` writes `"dev-pipeline@gronck-local": true` under `enabledPlugins` in
`~/.claude/settings.json`. The agents appear as `dev-pipeline:coder`, etc.,
and the skill as `/dev-pipeline:run-task`. Edits to the files take effect at the
next session or after `/reload-plugins`.

Later you can push `~/claude-plugins` to a private GitHub repo and add it with
`claude plugin marketplace add <owner>/<repo>` on other machines.

## Exclude one project

Inside that project:

```bash
claude plugin disable dev-pipeline@gronck-local --scope project   # committed: .claude/settings.json
# or, just for you, not committed:
claude plugin disable dev-pipeline@gronck-local --scope local     # .claude/settings.local.json
```

Both write `"dev-pipeline@gronck-local": false` under `enabledPlugins`; local >
project > user, so it overrides the user-scope `true`. To turn it back on:
`claude plugin enable dev-pipeline@gronck-local --scope project`.

## Override one agent in one project

Agent precedence (highest first): managed → `--agents` flag → project
`.claude/agents/` → user `~/.claude/agents/` → plugin `agents/`. A project file
with the same bare name shadows the plugin agent, so
`<project>/.claude/agents/tester.md` with `name: tester` replaces
`dev-pipeline:tester` there while the other four stay shared. Keep the
`SUCCESS:` / `BLOCKED:` output contract so `run-task` still parses it.

Note: plugin agents ignore the `hooks`, `mcpServers` and `permissionMode`
frontmatter fields; none of these agents uses them.

## Project contract (add to each project's CLAUDE.md)

Every agent reads this section first; `coder`, `tester`, `qa-mr` and `manager`
stop with `BLOCKED` when it is missing.

```markdown
## Pipeline contract

- **Base branch:** develop            <!-- task branches start here -->
- **PR target:** develop
- **Forge:** gh                     <!-- gh | glab | none (no PR step) -->
- **Branch naming:** task/NNNN-slug
- **Docs layout:** tasks docs/tasks/, plans docs/plan/, ADRs docs/adr/,
  living docs to keep current: docs/arch/overview.md. Docs language: English.
- **Test paths:** tests/            <!-- dirs or globs, e.g. **/*.test.ts -->
- **Gate commands:**
  - lint: `uv run ruff check <files>`
  - format: `uv run ruff format --check <files>`
  - types: `uv run mypy <files>`
  - tests: `uv run pytest`
- **Security-shaped paths:** app entrypoint, settings, anything with
  auth/token/credential/secret/password/session in the path, migrations/.
- **Merge/release policy:** PR opened without asking; merge, push to
  develop/main, release PRs, tags, force-push only on the owner's request.
  Never propose merging a PR whose CI is red or pending.
- **Extra rules:** never kill processes you didn't start (dev server on :8000);
  never print .env values; commit attribution line: <...>.
```

## Migration for currency_to_gold

1. Add a `## Pipeline contract` section to `CLAUDE.md` with these values:
   - Base branch and PR target `develop`; branches `task/NNNN-slug` (ADR 0009).
   - Docs layout `docs/tasks`, `docs/plan`, `docs/adr`, living
     `docs/arch/overview.md` and `docs/design.md` for UI; English.
   - Gate: `uv run ruff check`, `uv run ruff format --check`, `uv run mypy`
     (touched files), `uv run pytest`.
   - Security-shaped paths: `src/currency_to_gold/app.py`, any `settings.py`,
     auth/token/credential/secret/password/session paths, `security/`,
     `migrations/`, secret-like `.env.example` variables.
   - Merge/release policy: ADR 0008 + ADR 0010 (tags only on instruction, CI
     checked with `gh pr checks`).
   - Extra rules: `static/*.js` changes verified in real headless Chrome via the
     `tests/test_hints_browser.py` harness; dev server on :8000; no `timeout`
     binary (use `perl -e 'alarm 900; exec @ARGV' ...`); no RMT, no in-game
     automation; numbers shown to users come from code.
2. The existing "Agent pipeline" section of `CLAUDE.md` can shrink to a pointer
   at the plugin plus the contract (the ADRs stay as the decision record).
3. Remove `.claude/agents/{manager,coder,tester,code-reviewer,qa-mr}.md` and
   `.claude/skills/run-task/` — otherwise the project copies shadow the plugin
   (same bare names). Keep one only if this project needs a deliberate override.
4. Do it on a docs branch from `develop` with a PR, as any other change.

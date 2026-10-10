---
name: run-task
description: >
  Runs the manager -> coder -> tester || code-reviewer -> qa-mr dev pipeline for
  one task. Use when the user says "/run-task", "запусти задачу", "начни
  пайплайн для X", or asks to run the agent pipeline on a task. Args: a fresh
  task description (not yet recorded) or a reference to an existing task file.
---

Orchestrate the `dev-pipeline` agents: `manager`, `coder`, `tester`,
`code-reviewer`, `qa-mr`. You (the main thread) drive every step; the subagents
don't call each other or share your conversation, so every prompt is
self-contained.

**Agent names.** From this plugin the agents are `dev-pipeline:<name>`. A
project may shadow one with its own `.claude/agents/<name>.md` (same bare
name); then use the bare name. Use whatever the available-agents list shows.

## 0. Read the project contract

Read the project's `CLAUDE.md` and its `## Pipeline contract` section. You need:
Base branch, PR target, Forge (`gh`, `glab` or `none`), Branch naming, Docs
layout, Test paths, Gate commands, Security-shaped paths, Merge/release policy,
Extra rules. If the section is
missing, stop and tell the user: the pipeline needs it (template in the
plugin README). Don't guess.

## Steps

1. **Resolve input.** If the arg names an existing task file, go to step 3.
   Otherwise go to step 2.
2. **Discuss and record.** Invoke `manager` with the description. Continue the
   same instance via `SendMessage` for more discussion turns. Wait for the task
   slug(s). Owner decisions listed in the task go to the user; record their
   answers in the task file before implementing.
3. **Branch.** For each slug, update the contract's base branch (`git fetch` +
   fast-forward) and create the task branch from it (contract's naming,
   default `task/NNNN-slug`). The base ref for every later step is that commit.
4. **Implement.** Invoke `coder`: task/plan paths, branch, base ref, and on a
   retry the prior failure reason.
5. **Test and review in parallel.** On `coder` `SUCCESS`, launch `tester` and
   `code-reviewer` together. `code-reviewer` gets
   `git diff <base>...HEAD -- . <one ':(exclude)<glob>' per Test paths entry>`
   (e.g. `':(exclude)tests'`, `':(exclude)**/*.test.ts'`), is read-only and never
   runs the suite; name any security-shaped path from the contract that the
   diff touches, and the **review round**: 1 on the first pass, +1 on every
   later `code-reviewer` pass for this task, whatever triggered it (review
   findings, a `tester`/`qa-mr` BLOCKED, a deep review). Wait for both.
   **Review findings are mandatory** (the reviewer writes caveman-style, one
   line per finding):
   - Rounds 1 and 2: every finding (`🔴`, `🟡`, `🔵`, `❓`) goes back to a fresh
     `coder` call without asking the user (answer a `❓` yourself from the
     task/plan when you can; otherwise ask the user first), then step 5 again
     as the next round.
   - Round 3: `🔵 nit` findings are always skipped (the reviewer doesn't report
     them). Any other finding goes to the user, who decides per finding: fix
     (one more `coder` call + step 5) or follow-up task / accept.
6. **Gate.** Invoke `qa-mr` once `tester` says `SUCCESS` and `code-reviewer`
   says `REVIEW_CLEAN` (or the user has decided every round-3 finding). On
   `RECOMMEND_DEEP_REVIEW`, run one more `code-reviewer` pass over the full
   diff, tests included; its findings follow the same round rules.
7. **Retry on BLOCKED.** `BLOCKED` from `coder`, `tester` or `qa-mr` → fresh
   `coder` call with the reason, then every later step again. Cap: **3 retry
   cycles per task** (review rounds count toward it); a 4th `BLOCKED` stops the
   pipeline and goes to the user with a summary. If a subagent reports that the
   permission system refused an edit, don't make that edit yourself: put it to
   the user.
8. **Open the PR.** After `READY_FOR_MR` and a clean review, push the task
   branch and open the PR into the contract's PR target with `qa-mr`'s
   title/body and the contract's Forge (`gh pr create` / `glab mr create`),
   without asking. Forge `none`: stop after `READY_FOR_MR`, show the draft and
   leave the branch local. Report the link, a diff summary and any round-3
   findings the user chose to accept or defer.

## Always

- Merging, pushing directly to protected branches, release PRs, tags,
  force-pushing, and closing or deleting anything remote happen only on the
  user's explicit instruction (unless the contract's Merge/release policy says
  otherwise). Before proposing a merge, check the PR's CI (`gh pr checks`, `glab ci status`);
  never propose merging a red or pending PR.
- Commits only with an explicit pathspec; never `git add -A/./-u`.
- Report progress after each step (agent, SUCCESS/BLOCKED, retry count).

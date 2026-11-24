---
name: execute-fs-task
description: Implement one fullstack plan across {{CLIENT_REPO}} and {{SERVER_REPO}}, review it against the plan, then commit and open PRs only when acceptance criteria pass. Use when the user invokes /execute-fs-task or asks to execute a fullstack plan in docs/superpowers/plans.
disable-model-invocation: true
---

# Execute fullstack task

Run one plan file from `docs/superpowers/plans/` through implementation, review, and pull requests. The parent repo, `{{CLIENT_REPO}}`, and `{{SERVER_REPO}}` are separate git repositories.

Read `.cursor/rules/git-practice.mdc` and the superpowers `executing-plans` skill before starting. Follow `executing-plans` for how to read a plan and carry out its steps. This skill replaces that skill's finish step: do not use `finishing-a-development-branch`, do not commit during implementation, and do not ask whether to merge or open a PR.

Do not execute these plan files: {{EXCLUDED_PLANS}}.

## 1. Load the plan

Take the plan from the user prompt: a path under `docs/superpowers/plans/`, or a feature name that matches one file there.

Read the whole file. If a gap blocks implementation, stop and ask. Otherwise continue without further check-ins.

Sort every task by repository:

- Paths under `{{CLIENT_REPO}}/` are client work.
- Paths under `{{SERVER_REPO}}/` are server work.
- A task that lists both is both.

A repository with no tasks gets no branch and no implementer.

Branch slug: drop the date prefix and `.md` from the plan filename. `2026-11-09-categories.md` becomes `categories`.

## Workspace execution

The workspace is this repo, `{{CLIENT_REPO}}/`, `{{SERVER_REPO}}/`, and this project's `.cursor/` directory.

`.cursor/permissions.json` tells Auto-review to run in-workspace tool calls without an approval card. If that file is missing, write it before dispatching anyone:

```json
{
  "autoRun": {
    "allow_instructions": [
      "Allow every tool call whose files and effects stay inside this workspace, including {{CLIENT_REPO}} and {{SERVER_REPO}}: file reads and edits, {{PKG}} install, {{PKG}} lint, {{PKG}} build, {{PKG}} test, and local git commands (status, diff, log, add, commit, checkout, branch). Do not ask for approval.",
      "Allow sandboxed shell commands that only read and write inside the workspace."
    ],
    "block_instructions": [
      "Commands that push, force-push, or change git remotes should go through approval first.",
      "Commands that read or write outside this workspace should go through approval first."
    ]
  }
}
```

Parent, implementers, and the reviewer:

- Run shell in the sandbox. In-workspace reads, writes, `{{PKG}}` install, lint, build, test, and local git (`status`, `diff`, `log`, `add`, `commit`, `checkout`, `branch`) proceed without waiting.
- Do not set `request_smart_mode_approval`. Do not ask the user to click Allow for those calls.
- If Auto-review blocks an in-workspace command, change it so it stays inside the workspace sandbox and run it again. Do not escalate that retry.
- An approval card is only for work that cannot stay in the workspace: `git push`, `gh`, reading or writing outside the workspace, or a command the sandbox cannot run (Docker daemon, privileged ports).

Copy this section into every implementer and reviewer prompt. Subagents do not see this skill.

## 2. Create branches

In each repository that has work, and in the parent repo (the plan file always changes on success):

1. `git status` must be clean enough to branch. Untracked plan docs in the parent are fine. Stop if a repo has unrelated modifications.
2. Follow `.cursor/rules/git-practice.mdc`: in every repository you branch, check out `{{BASE_BRANCH}}` and run `git pull origin {{BASE_BRANCH}}` before creating the branch. Pull even when the user did not ask for a pull as a separate step.
3. Create `feature/<slug>` from that updated `{{BASE_BRANCH}}` before any feature edits. Same slug in every repo you branch.
4. If `feature/<slug>` already exists, stop and ask. Do not reset or reuse it.

Skip client or server when that side has no tasks.

## 3. Implement

Dispatch one `generalPurpose` subagent per side that has tasks. Run those subagents in parallel. Tasks inside one repository stay in order inside that subagent.

Pass `model: {{IMPLEMENTER_MODEL}}` on every implementer. The plan is already step-by-step, so a fast model follows it and spends fewer tokens. Do not inherit the parent model. Do not upgrade an implementer because a step looks important. If that model is rejected, use the closest available model named in the Task error and do not retry the rejected one.

Tell each implementer:

- Follow `executing-plans` for its tasks only: perform each step, write the tests the plan names, and run the verifications the plan names.
- Work only in its repository. Do not edit {{PROTECTED_FILES}} unless this plan says to.
- Do not commit, push, mark plan checkboxes, or switch branches.
- Stop and report BLOCKED when a step fails repeatedly or an instruction is unclear. Do not guess.

Wait for every implementer to finish. If any reports BLOCKED, show that report and stop. Do not review, mark checkboxes, commit, or open a PR.

## 4. Review

Dispatch one `generalPurpose` review subagent after implementation. Pass `model: {{REVIEWER_MODEL}}`. Review is the judgment step: missing criteria are more expensive than a cheaper reviewer. Do not inherit the parent model. If that model is rejected, use the closest available model of the same tier named in the Task error and do not retry the rejected one.

The reviewer compares the plan with the working trees.

The reviewer reads the plan's tasks, global constraints, and acceptance criteria, then reads the diffs in each touched app repo. It reports:

- **Missing**: acceptance criteria or task steps with no matching code or test
- **Refactor**: code that meets the plan but should change, with a concrete suggestion
- **Add**: tests or checks the plan requires that are absent, with a concrete suggestion

In each app repo that was branched, the reviewer also runs:

- `{{PKG}} lint`, `{{PKG}} build`, and `{{PKG}} test` when that script exists in `package.json`
- every verification command written in the plan, from the repo the plan names

A missing script, a failing command, a missing criterion, or any review concern is a failed review. Record command output with the concern.

## 5. Stop or mark the plan

If the review failed, show the reviewer's missing, refactor, and add items plus the command failures. Stop. Leave every `- [ ]` in the plan file unchanged. Do not commit or open a PR.

If every task step and acceptance criterion is done and lint, build, and tests passed, mark the executed plan file: change each `- [ ]` that belongs to that feature to `- [x]`. Do not edit other plan files.

## 6. Commit

Do this only after step 5 marked the plan.

Follow `.cursor/rules/git-practice.mdc` for the message: one concrete change per bullet, passed with a HEREDOC. One commit per repository.

- **Client or server**: stage only that feature's files. Omit secrets.
- **Parent**: stage only the executed plan file. Do not stage `.cursor/`, other plans, or submodule pointer updates.

Do not push or open a PR in this step. If a commit fails, report which repo succeeded and stop. Do not amend.

## 7. Open pull requests

Do this only after steps 1–6 all succeeded: the plan is marked, every required commit exists, and no earlier step stopped. If any of those failed, open no pull request.

In each repository that received a commit:

1. Push `feature/<slug>` with `-u`.
2. Open a pull request to `{{BASE_BRANCH}}` with `gh pr create`.

Use the same title in every repository for this feature. The title is `Implement <words> functionality`, where `<words>` is the branch slug with hyphens turned into spaces. `transactions` becomes `Implement transactions functionality`. `monthly-budgets` becomes `Implement monthly budgets functionality`.

The body has two sections, in this order:

```markdown
## Summary
- <one concrete change from this repository's commit>
- <next change>

## Test plan
- [x] <check that passed, with the command>
```

`## Summary` is the commit's bullet list for this repository: one concrete change per bullet, copied from that commit. Keep every bullet. Add a bullet when the diff shows a change the commit message omitted. For the parent repository, the summary is the plan-file update (checkboxes marked for this feature).

`## Test plan` lists the checks that passed in this repository (`{{PKG}} lint`, `{{PKG}} build`, `{{PKG}} test`, and any verification command from the plan). Mark each one `[x]`. For the parent repository, the test plan is that the executed plan file was marked.

Return every PR URL. If a push or PR fails, report which repo succeeded and stop. Do not open a pull request for the remaining repos. Do not force-push or amend.

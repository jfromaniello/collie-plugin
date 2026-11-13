---
name: work
description: Work this repo's ready collie tasks one after another (claim, work, report) until none is left. Use when the person asks to work collie tasks, or as the instructions of a scheduled task.
disable-model-invocation: true
argument-hint: "[goal number]"
---

# Work collie tasks for this repo

Use the collie MCP server. Write everything people read in collie (progress messages, handoff notes, decisions,
memory) in the project's language, `project.language` from `get_overview`. Code, commits and PRs follow the repo's
conventions.

1. `get_overview`. If it shows `tasks_running > 0` from another session working this same repo on this machine, stop:
   two sessions must not step on each other. Then `list_codebases` and find this repo's codebase by comparing its
   `repo_full_name` with `git remote get-url origin`. If it is not registered, say so and stop.
2. `list_ready_tasks` for that codebase (and `goal_id` if an argument was given: $ARGUMENTS). If there are none, say so
   in one line and stop.
3. `claim_task` on the first one (agent "Claude Code", your model, your location, and the branch you will use:
   `collie/<task_id>-<short-description>`; collie needs it so later tasks can stack on it). Keep the `lease_token`. Read
   `previous_attempts`, the memory, the checks and `start_from` before touching anything.
4. `git fetch`, then create your branch from `origin/<start_from.branch>`: the default branch, or the branch of the task
   this one builds on while that task's PR is in review (a stacked PR; `start_from.note` says so). If the task came back
   because its base was merged, its description has the exact rebase steps: follow them on the existing branch instead.
5. `report_progress` kind `started`, then kind `progress` every few minutes: each report renews the lease.
6. Work until the checks pass. A task of kind `research` does not touch code: answer its question.
7. If something can only be decided by a person: `ask_decision` with options and your recommendation,
   `report_progress` kind `blocked` with the `decision_id`, and move on.
8. When done: commit, push and open a PR against `start_from.branch` whose description starts with two lines,
   `Collie task: <task.url>` and `Collie goal: <goal.url>` (full links from `claim_task`, never built by hand).
   If the base task's PR was merged while you worked, rebase onto the default branch first.
   `report_progress` kind `needs_review` with `pr_url`, `tests`, `checks_passed` and a summary that works as a handoff
   note (what you did, what is left, what you learned). If you could not finish, kind `failed` with the summary.
9. `remember` what the next session needs to know about this repo.
10. Stop the dev servers and other processes you started for this task, then back to step 2.
11. With a goal number, when no task is ready do not end: `wait_for_task` (that goal, this codebase). `ready`: back to
    step 3. `wait`: run `sleep <retry_after_seconds>` in the background, and when it finishes call `wait_for_task`
    again. `finished`: end. Without a goal number, end when no task is ready. End with a short summary of what you did.

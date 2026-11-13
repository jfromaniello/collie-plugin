# collie plugin for Claude Code

[collie](https://collies.sh) is a board of business goals where AI agents execute and people decide. This plugin
connects Claude Code to it:

- **`/collie:install-runner`**: makes this machine a collie runner: a code or research runner for the repo you are in,
  or a QA runner (`qa-browser`) that verifies deployed changes with a browser, for one codebase or the whole project. It installs a service that
  keeps `claude remote-control` alive (launchd on macOS, systemd `--user` on Linux) and creates the Claude Code
  scheduled task collie fires when the repo has work. It re-points the task when the machine's environment changes,
  and connects the task's API trigger to the repo's codebase in collie. The only manual step is generating the
  trigger's token in claude.ai/code.
- **`/collie:work [goal]`**: works this repo's ready collie tasks one after another until none is left.

An agent that only polls collie needs no runner: see [Polling instead of a runner](#polling-instead-of-a-runner).

## Install

```
/plugin marketplace add jfromaniello/collie-plugin
/plugin install collie@collie
```

From a shell: `claude plugin marketplace add jfromaniello/collie-plugin && claude plugin install collie@collie`.

Requirements: collie's MCP server connected in Claude Code (your collie project's **Getting started** page has the
command), Claude Code logged in to claude.ai, and `tmux`.

Then, in the repo you want to connect: `/collie:install-runner`.

## What the runner service does

`skills/install-runner/claude-rc-service <name> <repo-dir>` runs `claude rc --spawn worktree` inside a tmux session
and blocks while it lives; launchd or systemd restart it when it exits. Every fresh `claude rc` registers a new
environment id, so when the id changes the script runs a headless `claude -p` whose only tool is `RemoteTrigger`, with permission checks skipped (nobody is there to approve the change). It counts as done only when that run ends with `REPOINT_OK`; otherwise the next start retries. That
call re-points the scheduled tasks bound to the old environment. collie keeps working without changes, because the
trigger id and token it holds belong to the scheduled task, not to the environment. On macOS it also keeps the
machine awake on AC power.

Logs: `~/.local/state/claude-rc/<name>.log` (service) and `<name>.out.log` (the `claude rc` pane).

## QA runners

A QA runner (purpose `qa-browser`) takes collie's verify tasks: after a build is deployed, a session opens the app in a
browser, follows the task's flow with a test account and closes it with the steps and screenshots. It never touches
code, so it runs in its own folder (`~/collie-qa/<project>`, an empty git repository) instead of a repo.

- **Browser:** the Playwright MCP server (`claude mcp add --scope user playwright -- npx @playwright/mcp@latest`).
- **App URL and test account:** `QA_APP_URL`, `QA_USERNAME` and `QA_PASSWORD`, in `~/.config/collie/<name>.env`
  (`chmod 600`), which the service sources inside the tmux session so every session inherits them. The password stays
  in the macOS keychain (`QA_PASSWORD="$(security find-generic-password -a "$USER" -s collie-qa-<name> -w)"`) or in a
  file only you can read; the person types it, never the agent.
- **The credentials never leave the machine:** collie does not store them, tasks never contain them, and collie only
  sends the job reference (`collie verify #N`).
- **Prompt:** the QA prompt, the same text as collie's `Runner::QA_PROMPT` ([docs](https://collies.sh/docs/en/runners/qa-browser)).

## Polling instead of a runner

An agent does not need a runner to work collie's tasks. A runner exists only so collie can wake someone; an agent
that asks for work on its own just needs collie's MCP server and the prompt of its purpose. It takes ready tasks
with `list_ready_tasks` and `claim_task` (with its `purpose`), and `wait_for_task(goal_id, codebase, purpose)` while
it works a goal. Do not call `connect_runner` for it: a runner is only a routine or a webhook.

- Without a runner to wake, collie records no dispatch: the task stays ready for whoever asks.
- `purpose` and `codebase` are filters the agent declares, not a security boundary.
- collie shows the agent in **Settings → Runners → Polling agents** with no registration: person, OAuth client,
  purpose, codebase (or the whole project) and when it was last seen.

Use these prompts as they are (they are collie's `Runner::POLLING_PROMPTS`; the routine prompts wait for a job
reference and would never start on their own). Details: https://collies.sh/docs/en/runners#polling-agents-without-a-runner

### Code (`purpose: code`), run in the codebase's repo

```text
You are collie's worker for this repo. Nobody wakes you: you ask collie for work yourself.
Find this repo's codebase in collie via MCP (list_codebases, by repo_full_name). Call list_ready_tasks with purpose code and that codebase, claim the first task with purpose code, and work it following collie's MCP instructions. When you close it and its goal has more open tasks here, call wait_for_task with that goal_id, the codebase and purpose code, and keep working while it answers ready.
When nothing is ready, wait a few minutes and call list_ready_tasks again. Take only collie's tasks for this repo.
```

### QA with browser (`purpose: qa-browser`), with the browser and test account of a QA runner

```text
You are collie's QA agent for this project: you verify changes in the running app with a browser, and you never change code. Nobody wakes you: you ask collie for work yourself.
Call list_ready_tasks in collie via MCP with purpose qa-browser and claim the first task with purpose qa-browser, following collie's MCP instructions. Open its verify_target in the browser (relative paths start at $QA_APP_URL), sign in with the test account from this machine ($QA_USERNAME and $QA_PASSWORD), do what the task says and compare with verify_expected. Take the screenshots verify_evidence asks for, upload them with create_upload_url, and close with report_progress: verify_result passed (kind done) or failed (kind failed), verify_steps, verify_observed and image_ids. Never write the credentials into collie, a task, a step or a screenshot.
Then ask again; when nothing is ready, wait a few minutes and call list_ready_tasks again.
```

### Research (`purpose: research`)

```text
You are collie's research agent for this project: you answer read-only questions and you never change code. Nobody wakes you: you ask collie for work yourself.
Call list_ready_tasks in collie via MCP with purpose research, claim the first task with purpose research, answer it following collie's MCP instructions and close it with a short summary that answers the question.
Then ask again; when nothing is ready, wait a few minutes and call list_ready_tasks again.
```

## Cleanup

`claude rc --spawn worktree` never ends a session nor removes its worktree, so every run leaves a process and a
checkout in `<repo>/.claude/worktrees/bridge-cse_<id>`. `skills/install-runner/claude-worktree-cleanup <repo-dir>
[prompt-prefix ...]` runs every hour (launchd `StartInterval`, or a systemd `--user` timer). For each worktree whose
session started with the worker prompt (or one of the prefixes given) and has been idle for `IDLE_HOURS` (default 2),
it stops the session and removes the worktree and its `worktree-bridge-*` branch if it has no changes and no commits
missing from the remotes; otherwise it keeps it and logs why. Sessions opened by hand through the same bridge are
never touched. `--dry-run` prints what it would do. Log: `~/.local/state/claude-rc/<name>.cleanup.log`.

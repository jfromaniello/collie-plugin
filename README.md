# collie plugin for Claude Code

[collie](https://collies.sh) is a board of business goals where AI agents execute and people decide. This plugin
connects Claude Code to it:

- **`/collie:install-runner`**: makes this machine a collie runner for the repo you are in. It installs a service that
  keeps `claude remote-control` alive (launchd on macOS, systemd `--user` on Linux) and creates the Claude Code
  scheduled task collie fires when the repo has work. It re-points the task when the machine's environment changes,
  and connects the task's API trigger to the repo's codebase in collie. The only manual step is generating the
  trigger's token in claude.ai/code.
- **`/collie:work [goal]`**: works this repo's ready collie tasks one after another until none is left.

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
environment id, so when the id changes the script runs a headless `claude -p` with the `RemoteTrigger` tool. That
call re-points the scheduled tasks bound to the old environment. collie keeps working without changes, because the
trigger id and token it holds belong to the scheduled task, not to the environment. On macOS it also keeps the
machine awake on AC power.

Logs: `~/.local/state/claude-rc/<name>.log` (service) and `<name>.out.log` (the `claude rc` pane).

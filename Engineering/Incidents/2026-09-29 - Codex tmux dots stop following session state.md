---
title: "Codex tmux dots stop following session state"
date: 2026-09-29
type: incident
status: "workaround implemented and regression-tested; existing clients need relaunch"
---

# 2026-09-29 — Codex tmux dots stop following session state

## Symptom

After updating Codex, tmux's yellow/running, red/waiting, and green/done dots no longer reliably followed individual Codex sessions. A live Codex client in pane `%2` had no pane state, while the shared server inherited `TMUX_PANE=%12`.

## Environment

- macOS, zsh, tmux; configuration managed with GNU Stow in `/Users/aperdomo/dotfiles`.
- Installed npm CLI: `codex-cli 0.158.0`.
- Running shared app-server: `0.159.0`, independently packaged under `~/.codex/packages/app-server-daemon/releases/`.
- Existing hooks: SessionStart, SessionEnd, UserPromptSubmit, PreToolUse, PostToolUse, PermissionRequest, Stop; all seven were enabled and trusted according to the server's read-only `hooks/list` API.
- Existing renderer and payload mapping were still compatible with the documented hook interface.

## Root Cause

The status setter targets `$TMUX_PANE`. The shared app-server runs several sessions in one process and retained the tmux environment of the client that launched it. Process inspection showed the daemon/updater using pane `%12` while another TUI used `%2`. Agent shell commands in the dotfiles session also inherited `%12`. The server reported multiple loaded root sessions. The evidence supports incorrect pane attribution through the shared process environment, rather than missing or untrusted hooks.

Two related problems were reproduced in isolated tests:

- zsh prompt and wrapper cleanup wrote the window-level `@agent_state` directly, erasing the aggregate for neighboring panes.
- The liveness sweep counted `codex app-server` processes as interactive agents, allowing a backend process to preserve stale state.

## Resolution

The smaller compatibility workaround was implemented:

1. `zsh/.zsh_aliases`: prepend `--no-daemon` to Codex launches inside tmux. Preserve explicitly supplied `--no-daemon` and `--remote` flags; keep argument boundaries and the original exit code.
2. Route shell and Claude/Codex wrapper cleanup through `agent-status.sh set clear`, which clears only the current pane and recomputes the window aggregate.
3. `tmux/.tmux/plugins/agent-status/scripts/agent-status.sh`: exclude Codex app-server/exec-server processes from interactive-agent liveness.
4. Document the limitation and activation procedure in repository `AGENTS.md`. Hook commands and trust hashes were not changed.

After finishing active turns, exit existing daemon-backed clients. In each existing tmux shell, run:

```sh
source ~/.zsh_aliases
codex resume
```

New shells load the wrapper automatically. Re-sourcing tmux alone does not replace already-running Codex clients.

**Tradeoff:** local tmux Codex sessions bypass the shared background server. Explicit remote connections and `command codex` bypass this workaround; environment-based dot routing is not reliable for shared-server sessions. Supporting that mode requires an explicit session-to-pane association.

## Validation and Remaining Rollout

`python3 tests/test_agent_status.py` runs nine regression checks using temporary isolated tmux servers and fake agent invocations/process tables. Six checks failed before the patch; all nine passed afterward. They cover pane aggregation, cleanup, daemon exclusion, child-process liveness, wrapper arguments, remote passthrough, and exit-code preservation.

Bash/zsh syntax and `git diff --check` passed. ShellCheck reported an existing SC2155 warning on the unchanged sweep timestamp declaration.

Existing interactive clients were not interrupted or restarted; verification of their display after relaunch remains a user rollout step. No real model call was used in the regression tests.

## Prevention / Runbook

- Inspect both `codex --version` and the running server version; updating the CLI and daemon are separate operations.
- Use `ps` ancestry plus selected `TMUX`/`TMUX_PANE` environment fields to diagnose attribution. Do not dump full environments containing credentials.
- Query `hooks/list` to distinguish a trust problem from a routing problem.
- Keep pane state (`@agent_pane_state`) and window aggregate (`@agent_state`) separate. Shell cleanup must use the setter.
- Preserve hook command paths and definitions unless intentionally changing the trust configuration.

## Related

- [Official Codex hooks documentation](https://learn.chatgpt.com/docs/hooks)
- [Codex changelog: 0.156.0 added the --no-daemon option](https://learn.chatgpt.com/docs/changelog)
- [Codex app-server protocol](https://learn.chatgpt.com/docs/app-server)
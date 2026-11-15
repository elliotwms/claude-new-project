---
name: hub-service
description: Keep the hub (the Claude Code Remote Control server in the projects folder) running on macOS with a launchd agent, and start, stop or check it with the claude-hub command. Use when asked to install, check, start, stop, pause, change or remove the hub or the hub service.
---

# Hub service

Installs a `claude-hub` command and a launchd agent. Every minute the agent runs `claude-hub ensure`, which starts the hub's tmux session if it isn't running and the user hasn't stopped it. The hub is the Remote Control server in `projects_dir` that the `new-project` skill describes; see the diagram there.

The agent doesn't run Claude Code itself. It starts the hub inside tmux, so the user can still `tmux attach` to it, and launchd only has to run a short command on a timer. This covers the hub's known ways of stopping: a crash, `claude remote-control` exiting after about ten minutes without network, and the machine restarting.

`claude-hub stop` pauses the hub: it stops the session and leaves a marker so the agent doesn't start it again, including after a reboot, until `claude-hub start`.

macOS only. On Linux, the same script works from a systemd user timer or a cron `@reboot` plus `*/1` entry running `claude-hub ensure`; say so and offer that instead.

## Config

Read `~/.config/new-project/config.json` if it exists (shared with `new-project`). Keys used here, all optional:

- `projects_dir`: where the hub runs. Default `~/projects`.
- `hub_session`: tmux session name for the hub. Default `hub`.

Values are written into the script when it's installed. To change them later, update the config and run this skill again.

## Install

1. **Check prerequisites.** Find absolute paths for `tmux` and `claude` with `command -v`. If either is missing, stop and say so. Confirm `projects_dir` exists.

2. **Write the command** to `~/.local/bin/claude-hub` and `chmod +x` it, substituting the absolute directories of `claude` and `tmux` into `PATH`, and the expanded `projects_dir` and `hub_session`. If an older `~/.local/bin/claude-hub-ensure` exists, remove it after step 4.
   ```bash
   #!/bin/bash
   # claude-hub: start, stop and check the Claude Code hub. Installed by the hub-service skill.
   # Usage: claude-hub [status|start|stop [--all]|ensure]
   export PATH="<claude dir>:<tmux dir>:/usr/bin:/bin"
   export LANG="${LANG:-en_US.UTF-8}"
   SESSION="<hub_session>"
   DIR="<projects_dir>"
   LABEL="local.claude-hub"
   PAUSED="$HOME/.local/state/claude-hub/paused"
   LOG="$HOME/Library/Logs/claude-hub.log"

   log() { echo "$(date -u +%FT%TZ) $*" >>"$LOG"; }
   running() { tmux has-session -t "=$SESSION" 2>/dev/null; }
   inside_hub() { [ -n "$TMUX" ] && [ "$(tmux display-message -p '#S')" = "$SESSION" ]; }

   # Start the hub unless it's running or paused. Run every minute by launchd.
   ensure() {
     [ -e "$PAUSED" ] && return 0
     running && return 0
     # Don't start into a dead network: Remote Control registers when it starts, and a server
     # that can't reach Anthropic exits after about ten minutes anyway. The next run retries.
     if ! curl -s -o /dev/null --max-time 5 https://api.anthropic.com; then
       log "offline, not starting $SESSION"
       return 0
     fi
     tmux new-session -d -s "$SESSION" -c "$DIR" \
       "claude remote-control --name '$SESSION' --spawn same-dir"
     log "started $SESSION in $DIR"
   }

   # Kill tmux sessions whose start directory is a project under $DIR (project servers).
   stop_projects() {
     tmux list-sessions -F '#{session_name}	#{session_path}' 2>/dev/null |
       while IFS=$'\t' read -r name path; do
         [ "$name" = "$SESSION" ] && continue
         case "$path" in "$DIR"/*) tmux kill-session -t "=$name" && echo "stopped $name" ;; esac
       done
   }

   case "${1:-status}" in
     ensure) ensure ;;
     start)
       rm -f "$PAUSED"
       log "start requested"
       # Prefer starting through launchd, so the hub runs in the logged-in session and can read
       # the keychain even when this command is run over SSH.
       if launchctl print "gui/$(id -u)/$LABEL" >/dev/null 2>&1; then
         launchctl kickstart "gui/$(id -u)/$LABEL"
       else
         ensure
       fi
       for _ in 1 2 3 4 5 6 7 8 9 10; do running && break; sleep 1; done
       if running; then echo "hub running (tmux attach -t $SESSION)"; else echo "hub not running yet; see $LOG"; fi
       ;;
     stop)
       mkdir -p "$(dirname "$PAUSED")" && touch "$PAUSED"
       log "stopped by user"
       [ "$2" = "--all" ] && stop_projects
       if inside_hub; then
         # Called from the hub itself: give it time to reply before its session ends.
         nohup sh -c "sleep 10; tmux kill-session -t '=$SESSION'" >/dev/null 2>&1 &
         echo "hub stopping in 10 seconds; start it again on the Mac or over SSH with: claude-hub start"
       else
         tmux kill-session -t "=$SESSION" 2>/dev/null
         echo "hub stopped; start it again with: claude-hub start"
       fi
       ;;
     status)
       if [ -e "$PAUSED" ]; then echo "hub: stopped (claude-hub start to resume)"
       elif running; then echo "hub: running in tmux session $SESSION"
       else echo "hub: not running; the agent tries again every minute (see $LOG)"; fi
       launchctl print "gui/$(id -u)/$LABEL" >/dev/null 2>&1 && echo "agent: loaded" || echo "agent: not loaded"
       tmux list-sessions -F '#{session_name}	#{session_path}' 2>/dev/null |
         while IFS=$'\t' read -r name path; do
           [ "$name" != "$SESSION" ] && case "$path" in "$DIR"/*) echo "project server: $name" ;; esac
         done
       ;;
     *) echo "usage: claude-hub [status|start|stop [--all]|ensure]" >&2; exit 2 ;;
   esac
   ```
   `--spawn same-dir` is right because `projects_dir` isn't itself a git repo.

3. **Write the launchd agent** to `~/Library/LaunchAgents/local.claude-hub.plist`, using absolute paths (launchd doesn't expand `~`):
   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>Label</key><string>local.claude-hub</string>
     <key>ProgramArguments</key>
     <array>
       <string>/Users/<user>/.local/bin/claude-hub</string>
       <string>ensure</string>
     </array>
     <key>RunAtLoad</key><true/>
     <key>StartInterval</key><integer>60</integer>
     <key>AbandonProcessGroup</key><true/>
     <key>ProcessType</key><string>Interactive</string>
     <key>LimitLoadToSessionType</key><string>Aqua</string>
     <key>StandardOutPath</key><string>/Users/<user>/Library/Logs/claude-hub.log</string>
     <key>StandardErrorPath</key><string>/Users/<user>/Library/Logs/claude-hub.log</string>
   </dict>
   </plist>
   ```
   - `AbandonProcessGroup` stops launchd from killing the tmux server when the command exits.
   - `ProcessType Interactive` stops macOS from throttling the hub while nobody is using the Mac. Without it, Remote Control sessions started under launchd have been reported to freeze at their first tool call until someone touches the keyboard or mouse, which is exactly when the user is away.
   - `LimitLoadToSessionType Aqua` runs it in the logged-in GUI session, where Claude Code can read its sign-in from the login keychain.

   Check it with `plutil -lint`.

4. **Load it.** If it's already loaded, unload it first:
   ```bash
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist 2>/dev/null
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist
   launchctl kickstart gui/$(id -u)/local.claude-hub
   ```
   If the hub is already running (for example, this skill is running inside it), `ensure` sees the session and does nothing. Never kill the hub's tmux session directly from inside the hub; use `claude-hub stop`.

5. **Make `claude-hub` easy to run.** If `~/.local/bin` isn't on the user's `PATH` in their shell profile, say so and give the line to add.

6. **Answer first-run prompts.** If the agent started a new hub, wait about 5 seconds and read `tmux capture-pane -pt <hub_session>`. If it shows `Enable Remote Control? (y/n)` or `Trust <dir>? [y/N]`, send `tmux send-keys -t <hub_session> y Enter` and check again. If it shows `Spawn mode for this project:`, send `1` then Enter (same-dir).

7. **Check what survives a reboot**, and report rather than change these (they need the user or `sudo`):
   - **Sleep**: `pmset -g | grep -E '^ *sleep '`. A value other than `0` means the machine will sleep and the hub goes offline. Fix: System Settings → Energy → Prevent automatic sleeping, or `sudo pmset -a sleep 0`.
   - **Automatic login**: `defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>/dev/null`. LaunchAgents start only after someone logs in, so without automatic login the hub won't return after a reboot until the user logs in. Automatic login isn't available while FileVault is on (`fdesetup status`).
   - **Power loss**: `pmset -g | grep autorestart`. `1` means the machine starts again after a power cut.

8. **Report**: the output of `claude-hub status`, anything from step 7 that will stop the hub coming back after a reboot, and the start and stop commands.

## Start, stop and check

- `claude-hub status`: whether the hub is running or stopped, whether the agent is loaded, and which project servers are running.
- `claude-hub stop`: stop the hub and keep it stopped, across reboots, until started again. Project servers keep running.
- `claude-hub stop --all`: also stop every project server (tmux sessions started in a folder under `projects_dir`).
- `claude-hub start`: clear the stop and start the hub now.

When the user asks the hub to stop itself, run `claude-hub stop` (or `stop --all`), then tell them it goes offline in about 10 seconds and can only be started again from the Mac or over SSH with `claude-hub start`. Ask first if they might want it back while away from the Mac.

## Troubleshooting

- Log: `tail -n 20 ~/Library/Logs/claude-hub.log` (a line per start or stop, plus any errors).
- `offline, not starting` lines: the machine couldn't reach `api.anthropic.com`; the hub starts on the first run after the network returns.
- Starts a minute apart: the hub is exiting straight away. Attach with `tmux attach -t <hub_session>`, or run the `claude remote-control` command by hand in `projects_dir`, to see why.
- A sign-in error usually means the tmux server was started from an SSH login, which can't read the keychain. `claude-hub start` avoids this by starting through launchd. To fix an existing one, stop the tmux server with `tmux kill-server` (this stops every tmux session, so ask first) and run `claude-hub start`.

## Remove

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist
rm ~/Library/LaunchAgents/local.claude-hub.plist ~/.local/bin/claude-hub
rm -rf ~/.local/state/claude-hub
```

This leaves a running hub alone. Stop it first with `claude-hub stop` if wanted.

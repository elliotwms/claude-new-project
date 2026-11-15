---
name: hub-service
description: Keep the hub (the Claude Code Remote Control server in the projects folder) running on macOS with a launchd agent, so it comes back after crashes, network drops and reboots. Use when asked to install, check, change or remove the hub service, or to keep the hub alive.
---

# Hub service

Installs a launchd agent that checks every minute whether the hub's tmux session exists and starts it if it doesn't. The hub is the Remote Control server in `projects_dir` that the `new-project` skill describes; see the diagram there.

The agent doesn't run Claude Code itself. It runs a short script that starts the hub inside tmux, so the user can still `tmux attach` to it, and launchd only has to run that script on a timer. This covers the hub's known ways of stopping: a crash, `claude remote-control` exiting after about ten minutes without network, and the machine restarting.

macOS only. On Linux, the same script works from a systemd user timer or a cron `@reboot` plus `*/1` entry; say so and offer that instead.

## Config

Read `~/.config/new-project/config.json` if it exists (shared with `new-project`). Keys used here, all optional:

- `projects_dir`: where the hub runs. Default `~/projects`.
- `hub_session`: tmux session name for the hub. Default `hub`.

Values are written into the script when it's installed. To change them later, update the config and run this skill again.

## Install

1. **Check prerequisites.** Find absolute paths for `tmux` and `claude` with `command -v`. If either is missing, stop and say so. Confirm `projects_dir` exists.

2. **Write the script** to `~/.local/bin/claude-hub-ensure` and `chmod +x` it, substituting the absolute directories of `claude` and `tmux` into `PATH`, and the expanded `projects_dir` and `hub_session`:
   ```bash
   #!/bin/bash
   # Starts the Claude Code hub in tmux if it isn't running. Installed by the hub-service skill.
   export PATH="<claude dir>:<tmux dir>:/usr/bin:/bin"
   export LANG="${LANG:-en_US.UTF-8}"
   SESSION="<hub_session>"
   DIR="<projects_dir>"

   tmux has-session -t "=$SESSION" 2>/dev/null && exit 0
   tmux new-session -d -s "$SESSION" -c "$DIR" \
     "claude remote-control --name '$SESSION' --spawn same-dir"
   echo "$(date -u +%FT%TZ) started $SESSION in $DIR"
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
       <string>/bin/bash</string>
       <string>/Users/<user>/.local/bin/claude-hub-ensure</string>
     </array>
     <key>RunAtLoad</key><true/>
     <key>StartInterval</key><integer>60</integer>
     <key>AbandonProcessGroup</key><true/>
     <key>LimitLoadToSessionType</key><string>Aqua</string>
     <key>StandardOutPath</key><string>/Users/<user>/Library/Logs/claude-hub.log</string>
     <key>StandardErrorPath</key><string>/Users/<user>/Library/Logs/claude-hub.log</string>
   </dict>
   </plist>
   ```
   - `AbandonProcessGroup` stops launchd from killing the tmux server when the script exits.
   - `LimitLoadToSessionType Aqua` runs it in the logged-in GUI session, where Claude Code can read its sign-in from the login keychain.

   Check it with `plutil -lint`.

4. **Load it.** If it's already loaded, unload it first:
   ```bash
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist 2>/dev/null
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist
   launchctl kickstart gui/$(id -u)/local.claude-hub
   ```
   If the hub is already running (for example, this skill is running inside it), the script sees the session and does nothing. Never kill the hub's tmux session from inside the hub.

5. **Answer first-run prompts.** If the script started a new hub, wait about 5 seconds and read `tmux capture-pane -pt <hub_session>`. If it shows `Enable Remote Control? (y/n)` or `Trust <dir>? [y/N]`, send `tmux send-keys -t <hub_session> y Enter` and check again. If it shows `Spawn mode for this project:`, send `1` then Enter (same-dir).

6. **Check what survives a reboot**, and report rather than change these (they need the user or `sudo`):
   - **Sleep**: `pmset -g | grep -E '^ *sleep '`. A value other than `0` means the machine will sleep and the hub goes offline. Fix: System Settings → Energy → Prevent automatic sleeping, or `sudo pmset -a sleep 0`.
   - **Automatic login**: `defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>/dev/null`. LaunchAgents start only after someone logs in, so without automatic login the hub won't return after a reboot until the user logs in. Automatic login isn't available while FileVault is on (`fdesetup status`).
   - **Power loss**: `pmset -g | grep autorestart`. `1` means the machine starts again after a power cut.

7. **Report**: whether the agent is loaded (`launchctl print gui/$(id -u)/local.claude-hub | grep -E 'state|last exit'`), whether the hub is running, and anything from step 6 that will stop it coming back after a reboot.

## Check

- Agent status: `launchctl print gui/$(id -u)/local.claude-hub`
- Log: `tail -n 20 ~/Library/Logs/claude-hub.log` (a line per start, plus any errors)
- Hub: `tmux capture-pane -pt <hub_session>`

If the log shows repeated starts a minute apart, the hub is exiting straight away. Attach with `tmux attach -t <hub_session>` or run the `claude remote-control` command by hand in `projects_dir` to see why. A sign-in error usually means the tmux server was started from an SSH login, which can't read the keychain; stop it with `tmux kill-server` (this stops every tmux session, so ask first) and let the agent start it again.

## Remove

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/local.claude-hub.plist
rm ~/Library/LaunchAgents/local.claude-hub.plist ~/.local/bin/claude-hub-ensure
```

This leaves a running hub alone. Stop it separately with `tmux kill-session -t <hub_session>`, and not from inside the hub.

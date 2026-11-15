---
name: new-project
description: Create a new project with a local git repo, optionally a GitHub repo set up from a template and a reference repo's settings, then start a Claude Code Remote Control server with worktrees for it. Use when asked to start or set up a new project.
---

# New project

Creates a project folder, writes a brief for it (or a full scaffold, if configured), commits it, optionally creates a GitHub repo for it, and starts a Claude Code Remote Control server for it in a detached tmux session, so the user can open worktree-isolated sessions for it from claude.ai/code or the Claude app.

Nothing about any one user's setup is hard-coded here. Personal choices live in a config file (below).

## The pattern

This skill is usually run by the **hub**: a Claude Code Remote Control server in `projects_dir`, in its own tmux session (`hub_session`, default `hub`). The user opens the hub from their phone or browser to create projects and to start or stop project servers. Each project gets its own Remote Control server in a tmux session named after the project, and each session the user opens there gets its own git worktree. The hub doesn't direct project work; the user talks to each project's sessions directly.

Why a server per project: the Claude app lists each machine running `claude remote-control` as a device card, and when the user starts a session from it, the folder picker offers only the folders where a server is running. A project appears there, with its branch, once its server is up, and `--spawn worktree` gives each of its sessions its own worktree.

The hub needs something to keep it running. The `hub-service` skill does this on macOS with launchd; any process supervisor that keeps `claude remote-control` running in `projects_dir` inside a tmux session named `hub_session` works the same way, and this skill doesn't depend on which one is used. Tools that keep an interactive `claude --remote-control` session alive (one conversation) aren't a substitute: the hub has to run in server mode for the device card and folder picker.

```
               You: Claude app or claude.ai/code
                 │                         │
                 │ Remote Control          │ Remote Control
                 ▼                         ▼
 ┌──────────────────────────┐        ┌───────────────────────────────┐
 │ Hub                      │creates,│ Project server: foo           │
 │ tmux "hub"               │starts, │ tmux "foo"                    │
 │ cwd: ~/projects          ├───────▶│ cwd: ~/projects/foo           │
 │ remote-control           │ stops  │ remote-control                │
 │   --spawn same-dir       │        │   --spawn worktree            │
 │                          │        │   ├─ session 1 → worktree a   │
 └──────────▲───────────────┘        │   └─ session 2 → worktree b   │
            │ restarts if it dies    └───────────────────────────────┘
 ┌──────────┴───────────────┐        ┌───────────────────────────────┐
 │ supervisor (hub-service) │        │ Project server: bar  …        │
 └──────────────────────────┘        └───────────────────────────────┘
```

When this skill runs inside the hub, never stop or restart the hub's own tmux session: the skill is running inside it. If the user asks to stop the hub, use `claude-hub stop` (see the `hub-service` skill), which allows for this.

## Config

Read `~/.config/new-project/config.json` if it exists. Every key is optional:

```json
{
  "projects_dir": "~/projects",
  "github": "ask",
  "github_owner": "",
  "visibility": "private",
  "template_repo": "",
  "settings_from": "",
  "managed_by": "",
  "spawn": "worktree",
  "permission_mode": "acceptEdits",
  "hub_session": "hub",
  "scaffold": "brief"
}
```

- `projects_dir`: parent folder for projects. Default `~/projects`.
- `github`: `ask` (default), `always` or `never`.
- `github_owner`: user or org that owns new repos. Default: the account `gh` is logged in as.
- `visibility`: `private` (default) or `public`. The user's request overrides it.
- `template_repo`: `owner/repo` of a GitHub template repo to create new repos from. Its files (workflows, issue templates, etc.) are the project's starting point.
- `settings_from`: `owner/repo` of a reference repo whose GitHub settings, rulesets and labels new repos copy. Templates copy files, never settings, so this is how settings are kept consistent.
- `managed_by`: `owner/repo` of an infrastructure-as-code repo (Terraform or similar) that manages the user's GitHub repos. When set, the skill doesn't create or configure the GitHub repo itself; it opens a PR there instead (step 4b).
- `spawn`, `permission_mode`: passed to `claude remote-control` for project servers.
- `hub_session`: tmux session name of the hub. Default `hub`. No project may use this name.
- `scaffold`: how much of the project to set up before handing over. `brief` (default) writes the user's description into the project and stops, so design and development start in the project's own sessions. `full` also scaffolds the app for its stack and builds it once. The user's request overrides it ("and scaffold it", "just the repo").

If the file doesn't exist, use the defaults and mention once in the report that the user can create it to set these.

## Inputs

- **Name**: from the arguments or the request. A short lowercase slug (letters, digits, hyphens). If none was given, ask.
- **Kind and purpose**: what the project is (iOS app, Go service, etc.) and a line on what it does.
- **GitHub**: whether to create a GitHub repo, unless config `github` is `always` or `never`.

If any of these are missing, ask for all of them in one short question before starting.

## Steps

1. **Check the name is free.** If the name equals `hub_session`, ask for another. If `<projects_dir>/<name>` exists, or `tmux has-session -t <name>` succeeds, stop and ask whether to reuse it or pick another name. If GitHub is wanted, also run `gh repo view <owner>/<name>`; if it exists, stop and ask.

2. **Check prerequisites.** `tmux` and `claude` must be on PATH. If GitHub is wanted, `gh auth status` must succeed; if it doesn't, carry on without GitHub and say in the report that `gh auth login` fixes it. With `scaffold: full` for an iOS project, `xcodebuild -version` must succeed; if it doesn't, stop and say Xcode is needed.

3. **Create the project.**
   - With a `template_repo` and GitHub wanted (and no `managed_by`): `gh repo create <owner>/<name> --<visibility> --template <template_repo> --clone` from inside `projects_dir`.
   - Otherwise: create `<projects_dir>/<name>` and `git init -b main`.

   Then, depending on `scaffold`:

   - **`brief`** (default): don't write any app code or choose its structure; that work belongs to the project's own sessions. Write:
     - `CLAUDE.md` with a `## Brief` section holding the user's description of the project in their own words (lightly tidied, nothing invented), the stack if they named one, and a line saying nothing has been built yet, so the first session should start by planning with the user. Keep anything that came from a template.
     - `README.md` with the project name and a one-line description, unless a template provided one.
     - A `.gitignore` for the stack if the user named one.
   - **`full`**: as `brief`, then scaffold for the stack, keeping it minimal and conventional for that ecosystem, and add the exact build, test and run commands to `CLAUDE.md`. For an iOS app, prefer a generated project (XcodeGen `project.yml`, with the `.xcodeproj` gitignored) over a committed `.xcodeproj`, SwiftUI, and a local Swift package for logic with tests; confirm it builds for the simulator. If the project has a build, run it once and note whether it passed.

   Commit. Worktree mode needs at least one commit, so this must happen before step 5.

4. **GitHub (only if wanted).**

   **4a. No `managed_by`:**
   - If the repo wasn't made from a template in step 3: `gh repo create <owner>/<name> --<visibility> --source . --remote origin --push --description "<one-line purpose>"`. If it was, `git push`.
   - If `settings_from` is set, copy its configuration. Do each part separately; if one fails (for example rulesets on a private personal repo need a paid plan), note it and carry on.
     - **Settings**: read `gh api repos/<settings_from>` and PATCH the new repo with the same values for: `allow_merge_commit`, `allow_squash_merge`, `allow_rebase_merge`, `allow_auto_merge`, `allow_update_branch`, `delete_branch_on_merge`, `squash_merge_commit_title`, `squash_merge_commit_message`, `merge_commit_title`, `merge_commit_message`, `has_issues`, `has_wiki`, `has_projects`, `has_discussions`, `web_commit_signoff_required`.
     - **Rulesets**: list `gh api repos/<settings_from>/rulesets` and copy only those with `source_type: Repository` (organization rulesets already apply wherever they apply, and can't be copied into a repo). For each, GET it and POST `name`, `target`, `enforcement`, `conditions`, `rules` and `bypass_actors` to `repos/<owner>/<name>/rulesets`. Keep `RepositoryRole` and `OrganizationAdmin` bypass actors; drop team, app or user actors whose IDs may not exist for the new repo, and report them. For a `required_status_checks` rule, keep only checks the new repo will actually report (from its own or its template's workflows); drop the rest and report them, because a required check that never runs blocks every merge.
     - **Labels**: `gh label clone <settings_from> --repo <owner>/<name>`.
     - **Branch protection**: if the reference repo uses classic branch protection rather than rulesets, say so in the report and don't copy it.
   - If the reference repo's rules now block direct pushes to the default branch, any further commits from this skill go through a branch and PR.

   **4b. With `managed_by`:** don't create or configure the repo directly. Clone `managed_by`, find how existing repos are declared (for example `github_repository` resources or a module), and open a PR adding this one following the same pattern, using `template_repo` if the pattern supports it. Then stop GitHub work, keep the local repo, and tell the user to merge and apply that PR, after which they can add the remote with `git remote add origin` and push. Never run `terraform apply` or equivalent.

5. **Start the server** in a detached tmux session named after the project:
   ```bash
   tmux new-session -d -s <name> "cd <projects_dir>/<name> && claude remote-control --name <name> --spawn <spawn> --permission-mode <permission_mode>"
   ```
   `--spawn worktree` needs a git repo with at least one commit, which step 3 guarantees.

6. **Answer first-run prompts.** Wait about 5 seconds, then read the pane with `tmux capture-pane -pt <name>`. If it shows `Trust <dir>? [y/N]` or `Enable Remote Control? (y/n)`, send `tmux send-keys -t <name> y Enter` and check again. If it shows `Spawn mode for this project:`, send `2` then Enter for worktree, or `1` for same-dir, to match `spawn`. Repeat until the pane shows the server waiting for connections, or an error.

7. **Report back** in a few lines: the project path; the GitHub repo URL, or the infra PR URL, or that GitHub was skipped and why; which reference settings applied and which didn't; with `scaffold: full`, whether the build passed; and that the project's folder now appears in the Claude app when starting a session from this machine's device card, which is where to start designing and building it. If the server failed, include the error text from the pane.

## Managing servers

- List running servers: `tmux ls`
- Restart one: `tmux kill-session -t <name>`, then steps 5 and 6.
- Stop one: `tmux kill-session -t <name>`.

# new-project

A Claude Code plugin that starts a new project end to end. Ask Claude for a new project and it:

1. creates the folder and scaffolds it for the stack you name;
2. makes a git repo with an initial commit;
3. optionally creates a GitHub repo, from a template and with settings copied from a repo you already like;
4. starts a [Remote Control](https://code.claude.com/docs/en/remote-control) server for it in tmux, with each session in its own git worktree.

A minute later the project shows up in your session list at claude.ai/code and in the Claude app, so you can work on it from your phone while it runs on your own machine.

It's built for an always-on machine, such as a Mac mini, running a Claude Code session you reach through Remote Control. Ask that session for a new project and you never need to sit at the machine.

## Requirements

- [Claude Code](https://code.claude.com) signed in with a claude.ai plan that includes Remote Control (Pro, Max, Team or Enterprise; API keys don't work)
- `git` and `tmux`
- [`gh`](https://cli.github.com), logged in with `gh auth login`, if you want GitHub repos
- For iOS projects: Xcode, and [XcodeGen](https://github.com/yonaskolb/XcodeGen) (`brew install xcodegen`)

## Install

In Claude Code:

```
/plugin marketplace add elliotwms/claude-new-project
/plugin install new-project@claude-new-project
```

Update later with `/plugin marketplace update claude-new-project`.

## Use

Ask in plain words, for example:

> New project called `tide-times`: an iOS app showing tide times for the Welsh coast. Put it on GitHub.

Or run the skill directly with `/new-project:new-project tide-times`. If you leave out the name, the kind of project or whether you want GitHub, Claude asks once before starting.

## Configure

Copy `config.example.json` to `~/.config/new-project/config.json` and edit it. Every key is optional; without the file the defaults apply.

| Key | Default | What it does |
|---|---|---|
| `projects_dir` | `~/projects` | Where new projects go |
| `github` | `ask` | `ask`, `always` or `never` create a GitHub repo |
| `github_owner` | your `gh` login | User or org that owns new repos |
| `visibility` | `private` | `private` or `public`; your request overrides it |
| `template_repo` | none | `owner/repo` template to create repos from (files only) |
| `settings_from` | none | `owner/repo` whose settings, rulesets and labels new repos copy |
| `managed_by` | none | `owner/repo` of the Terraform (or similar) repo that manages your GitHub repos |
| `spawn` | `worktree` | Remote Control spawn mode: `worktree` or `same-dir` |
| `permission_mode` | `acceptEdits` | Starting permission mode for remote sessions |

### Keeping repos consistent

GitHub template repos copy files but not settings, rulesets or labels. To keep those consistent, set `settings_from` to a repo that's already set up the way you like. The skill copies its merge settings, features, repo-level rulesets and labels to each new repo. It leaves out anything that wouldn't work on the new repo and lists it in its report:

- organization rulesets, which already apply where they apply
- bypass permissions for specific teams, apps or users
- required status checks the new repo doesn't run, which would otherwise block every merge

Rulesets on private repos owned by a personal account need GitHub Pro. On the free plan those steps fail and the rest still apply.

If you manage GitHub repos with Terraform, set `managed_by` instead. The skill then opens a PR in that repo, following the pattern it already uses, and leaves applying it to you. It never runs `terraform apply`.

## Managing servers

Each project's server runs in a tmux session named after the project.

- `tmux ls` lists them
- `tmux attach -t <name>` shows one (detach with Ctrl-b, then d)
- `tmux kill-session -t <name>` stops one

Servers stop if the machine restarts, and a server exits after about ten minutes without a network connection. Ask Claude to restart one, or rerun the command from the skill.

## Licence

[MIT](LICENSE)

# claude-code-workspaces

A Claude Code plugin for managing parallel development with git worktrees. Each task gets its own isolated workspace — named, tracked, and easy to switch between.

## Why

Working on multiple tasks in parallel with Claude Code means juggling branches manually. This plugin automates it: create a workspace, get a named worktree, and Claude tracks it across sessions.

## Install

```
/plugin marketplace add gcassarini/claude-code-workspaces
/plugin install claude-code-workspaces@gcassarini-claude-code-workspaces
```

## Usage

All commands start with `/workspace`.

### Create a workspace

```
/workspace new
```

Creates a new git worktree with a random `color-animal` name (e.g. `cobalt-narwhal`), sets up the branch `ws/cobalt-narwhal`, and drops you into it.

### List workspaces

```
/workspace
/workspace list
```

Shows all active workspaces for the current repo with branch, description, and uncommitted changes.

### Join a workspace

```
/workspace red-fox
```

Switches into an existing workspace. If it has a description set, Claude asks for a session topic to name the session `auth-refactor/review`.

### Describe a workspace

```
/workspace describe "auth-refactor"
```

Sets a human-readable description on the current workspace. Updates the session name.

### Archive / restore

```
/workspace archive [name]     # detach worktree, keep branch + metadata
/workspace unarchive <name>   # restore worktree from archived state
```

### Delete

```
/workspace delete <name>
```

Deletes the worktree, local branch, and remote branch. Asks for confirmation first.

### Branch overview

```
/workspace branches
```

Table of all workspace branches (active + archived) with merged status and ahead/behind counts.

### Cleanup merged branches

```
/workspace cleanup
```

Finds all merged workspace branches and removes them after confirmation.

### Other commands

```
/workspace rename <new-branch>   # rename current branch, update metadata + remote
/workspace status                # details on the current workspace
/workspace prune                 # clean stale worktree entries
```

## How it works

Workspaces are stored under `~/.claude-code-workspaces/<repo-name>/active/`. Each workspace is a standard git worktree with a `.workspace.json` metadata file that tracks the branch name as source of truth (survives manual renames).

```
~/.claude-code-workspaces/
└── my-repo/
    ├── active/
    │   └── cobalt-narwhal/
    │       ├── .workspace.json
    │       └── ... (git worktree)
    └── archived/
        └── red-fox/
            └── .workspace.json
```

## Per-repo setup hook

Create `.claude/claude-code-workspaces.sh` in your repo to run custom setup after each new workspace is created:

```bash
#!/bin/bash
# $1 = workspace name (e.g. cobalt-narwhal)
# $2 = branch name   (e.g. ws/cobalt-narwhal)
# $3 = original repo path

cp .env.example .env
npm install
```

## Session naming

| Situation | Session name |
|-----------|-------------|
| New workspace | `cobalt-narwhal` |
| After `/workspace describe "auth-refactor"` | `auth-refactor` |
| Joining with description set | `auth-refactor/review` |

## License

MIT

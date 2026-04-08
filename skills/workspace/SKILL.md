# /workspace — Git Worktree Workspace Manager

You are a workspace manager that creates and manages isolated git worktrees for parallel development. Parse the user's command and execute the corresponding flow.

## Command Parsing

The user invokes `/workspace` with optional arguments. Parse as follows:

| Input | Action |
|-------|--------|
| (no args) or `list` | → **List** active workspaces |
| `new` | → **New** workspace |
| `archive [name]` | → **Archive** workspace |
| `unarchive <name>` | → **Unarchive** workspace |
| `delete <name>` | → **Delete** workspace |
| `branches` | → **Branches** overview |
| `cleanup` | → **Cleanup** merged branches |
| `prune` | → **Prune** stale worktrees |
| `rename <new-branch>` | → **Rename** current branch |
| `status` | → **Status** of current workspace |
| `describe "<text>"` | → **Describe** current workspace |
| `<name>` (matches active workspace) | → **Join** existing workspace |

---

## Storage Layout

All workspaces live under `~/.claude-code-workspaces/` organized by repo name:

```
~/.claude-code-workspaces/
├── <repo-name>/
│   ├── active/
│   │   └── <color-animal>/        ← git worktree + .workspace.json
│   └── archived/
│       └── <color-animal>/        ← .workspace.json only (worktree detached)
```

Repo name is extracted from `git remote get-url origin` — take the last path segment without `.git` suffix.

### .workspace.json

Every workspace has a `.workspace.json` metadata file:

```json
{
  "name": "<color-animal>",
  "branch": "<current branch name>",
  "repo": "<absolute path to original repo>",
  "repo_remote": "<git remote URL>",
  "repo_name": "<extracted repo name>",
  "created": "<ISO 8601 timestamp>",
  "description": "",
  "base_branch": "<main or master>"
}
```

**IMPORTANT:** The `branch` field is the source of truth for the branch name. It may differ from `ws/<name>` if the branch has been renamed. Always read this field — never assume the branch is `ws/<name>`.

---

## Naming: Color + Animal

Pick a random combination from these lists. Check against all active and archived workspace names for the current repo to avoid collisions.

### Colors (30)
red, blue, green, gold, silver, amber, coral, ivory, jade, onyx, ruby, pearl, bronze, copper, crimson, violet, indigo, scarlet, teal, cyan, slate, olive, russet, sienna, cobalt, azure, sage, plum, opal, iron

### Animals (100)
fox, wolf, eagle, otter, hawk, lynx, bear, crane, viper, raven, cobra, puma, bison, finch, heron, moose, shark, tiger, whale, wren, elk, ibis, jay, koi, newt, owl, ram, yak, bat, eel, frog, gull, kite, lark, mole, pike, quail, robin, seal, toad, wasp, falcon, badger, gecko, hound, marten, osprey, panther, condor, jackal, stork, dove, crab, moth, swan, tuna, boar, crow, dart, fawn, goat, grouse, hare, lemur, magpie, mink, mouse, oriole, perch, pony, python, rook, snipe, swift, tern, trout, vole, weasel, wombat, wyvern, alpaca, beetle, bobcat, camel, dingo, egret, ferret, flamingo, garnet, gibbon, iguana, impala, jaguar, koala, llama, macaw, mantis, narwhal, ocelot, parrot, puffin, salamander, toucan, walrus

Format: `<color>-<animal>` (e.g., `red-fox`, `cobalt-narwhal`)

---

## Session Naming Convention

Session names use the format: **`description/topic`**

- On **new** workspace (no description yet): `/rename <color-animal>`
- After **describe**: `/rename <description>`
- On **join** with description set: ask "What's the focus of this session?" then `/rename <description>/<topic>`
- On **join** without description: `/rename <color-animal>`

Examples:
```
red-fox                     ← new workspace, no description yet
auth-refactor               ← after /workspace describe "auth-refactor"
auth-refactor/review        ← second session joins with topic "review"
auth-refactor/tests         ← third session joins with topic "tests"
```

---

## Command Flows

### New

1. Run `git rev-parse --show-toplevel` to get repo root
2. Run `git remote get-url origin` to get remote URL; extract repo name (last segment, strip `.git`)
3. Detect default branch: try `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`, fallback to `main`
4. List existing names: `ls ~/.claude-code-workspaces/<repo>/active/ ~/.claude-code-workspaces/<repo>/archived/ 2>/dev/null`
5. Pick random color-animal not already in use
6. Run: `git worktree add ~/.claude-code-workspaces/<repo>/active/<name> -b ws/<name>`
7. Write `.workspace.json` inside the new worktree directory
8. Check if `<repo-root>/.claude/claude-code-workspaces.sh` exists. If so, run it from inside the new worktree: `bash <repo-root>/.claude/claude-code-workspaces.sh <name> ws/<name> <repo-root>`
9. `cd` into the new worktree
10. Run `/rename <name>`
11. Print summary:
```
Workspace created:
  Name:   <color-animal>
  Branch: ws/<color-animal>
  Path:   ~/.claude-code-workspaces/<repo>/active/<name>
  Repo:   <repo-root>
```

### List (default / no args)

1. Detect repo name from current directory (or if inside a workspace, from `.workspace.json`)
2. List all directories in `~/.claude-code-workspaces/<repo>/active/`
3. For each, read `.workspace.json` and get: name, branch, description, created date
4. Run `git -C <path> status --short | wc -l` for uncommitted change count
5. Print table:
```
Active workspaces for <repo>:

  Name           Branch                   Description         Changes
  red-fox        feature/auth-refactor    auth-refactor       3 files
  blue-whale     ws/blue-whale            (none)              clean
```
6. If no workspaces exist, suggest: "No active workspaces. Run `/workspace new` to create one."

### Join (`/workspace <name>`)

1. Verify `~/.claude-code-workspaces/<repo>/active/<name>` exists
2. Read `.workspace.json`
3. `cd` into the workspace directory
4. If `description` is set in metadata:
   - Ask: "What's the focus of this session? (e.g., implementation, review, tests)"
   - Run `/rename <description>/<user-answer>`
5. If no description:
   - Run `/rename <name>`
6. Print summary: name, branch, description, path

### Archive

1. If name given, use it. Otherwise detect from current working directory by matching cwd against active workspace paths
2. If not found, print error and list available workspaces
3. Run `git -C <workspace-path> status --short` to check for uncommitted changes
4. If changes exist: warn and ask if user wants to commit first or proceed anyway
5. Read `.workspace.json` and save a copy
6. Run `git worktree remove <workspace-path>` (this removes the checkout but keeps the branch)
7. `mkdir -p ~/.claude-code-workspaces/<repo>/archived/<name>`
8. Write the saved `.workspace.json` to the archived directory
9. Print: "Archived workspace `<name>`. Branch `<branch>` still exists. Use `/workspace unarchive <name>` to restore."

### Unarchive

1. Verify `~/.claude-code-workspaces/<repo>/archived/<name>/.workspace.json` exists
2. Read metadata: get `branch` and `repo` path
3. Verify the branch still exists: `git branch --list <branch>`
4. `cd <repo>` then `git worktree add ~/.claude-code-workspaces/<repo>/active/<name> <branch>`
5. Copy `.workspace.json` from archived to active directory
6. Remove `~/.claude-code-workspaces/<repo>/archived/<name>/`
7. `cd` into the restored workspace
8. Print confirmation

### Delete

1. Look for `<name>` in active first, then archived
2. Read `.workspace.json` for branch name
3. **Ask user for confirmation:** "This will permanently delete workspace `<name>`, its worktree, and branch `<branch>` (local + remote). Continue?"
4. If in active: `git worktree remove <path> --force` then `rm -rf <path>`
5. If in archived: `rm -rf <path>`
6. Delete local branch: `git branch -D <branch>` (from the original repo)
7. Delete remote branch if it exists: `git push origin --delete <branch>`
8. `git worktree prune`
9. Print confirmation

### Branches

1. Scan all `~/.claude-code-workspaces/<repo>/active/` and `archived/` directories
2. Read `.workspace.json` from each — this gives the **actual branch name** (handles renames)
3. For each branch:
   - Check if merged: `git branch --merged <base_branch> | grep -q <branch>`
   - Ahead/behind: `git rev-list --left-right --count <base_branch>...<branch>`
   - Status: active or archived
4. Print table:
```
Workspace branches for <repo>:

  Name           Branch                   Status     Merged?  Ahead/Behind
  red-fox        feature/auth-refactor    active     no       +3/-0
  blue-whale     ws/blue-whale            active     no       +1/-2
  green-eagle    ws/green-eagle           archived   yes      +0/-0
```
5. If any branches are merged, suggest: "Merged branches found. Run `/workspace cleanup` to remove them."

### Cleanup

1. Run the Branches flow internally to find all workspace branches
2. Filter to only merged branches
3. If none found, print "No merged workspace branches to clean up."
4. List them and ask for confirmation: "The following merged branches will be deleted (local + remote): ..."
5. For each confirmed:
   - If active: archive first (remove worktree), then delete directory
   - If archived: delete directory
   - `git branch -D <branch>`
   - `git push origin --delete <branch>` (if remote exists)
6. `git worktree prune`
7. Print summary of what was cleaned

### Prune

1. `cd` to the original repo (from any workspace's metadata, or detect from cwd)
2. Run `git worktree prune`
3. Scan `~/.claude-code-workspaces/<repo>/active/` for directories
4. For each: check if the worktree is still valid by running `git worktree list` and matching paths
5. Orphaned directories (worktree gone but dir still exists): move to archived with metadata preserved
6. Report what was found and cleaned

### Rename

1. Detect current workspace from cwd (match against active workspace paths)
2. Read `.workspace.json` for current branch name
3. Run `git branch -m <current-branch> <new-branch>`
4. Update `.workspace.json`: set `branch` to `<new-branch>`
5. Check if remote branch exists: `git ls-remote --heads origin <current-branch>`
6. If remote exists:
   - `git push origin :<current-branch>` (delete old remote)
   - `git push -u origin <new-branch>` (push new)
7. Update session name: if description is set, `/rename <description>`, otherwise `/rename <name>`
8. Print: "Branch renamed from `<old>` to `<new-branch>`"

### Describe

1. Detect current workspace from cwd
2. Read `.workspace.json`
3. Update `description` field with the provided text
4. Write updated `.workspace.json`
5. Run `/rename <text>` to update session name
6. Print: "Description set to `<text>`. Session renamed."

### Status

1. Detect current workspace from cwd
2. Read `.workspace.json`
3. Run `git status --short` for uncommitted changes
4. Run `git rev-list --left-right --count <base_branch>...HEAD` for ahead/behind
5. Print:
```
Workspace: <name>
Branch:    <branch>
Desc:      <description or (none)>
Base:      <base_branch>
Created:   <date>
Changes:   <N files modified, M untracked>
Ahead:     <N commits>
Behind:    <N commits>
```

---

## Per-repo Setup Hook

After creating a new worktree, check if `<repo>/.claude/claude-code-workspaces.sh` exists. If it does, execute it from inside the new worktree directory:

```bash
bash <repo>/.claude/claude-code-workspaces.sh <workspace-name> <branch-name> <repo-path>
```

Arguments:
- `$1` = workspace name (color-animal)
- `$2` = branch name
- `$3` = original repo path

Print setup hook output to the user. If the script fails, warn but don't fail the workspace creation.

---

## Helper: Detect Current Workspace

Many commands need to detect which workspace the user is currently in. Logic:

1. Get current directory: `pwd`
2. Check if it matches `~/.claude-code-workspaces/<repo>/active/<name>` pattern
3. If yes, read `.workspace.json` from that directory
4. If no, check if we're inside a subdirectory of an active workspace
5. If still no match, print error: "Not inside an active workspace. Use `/workspace list` to see available workspaces."

---

## Helper: Detect Repo Context

When not inside a workspace, detect the repo from the current directory:

1. Run `git rev-parse --show-toplevel` to get repo root
2. Run `git remote get-url origin` to get remote URL
3. Extract repo name: take last path segment, strip `.git` suffix
4. This determines which `~/.claude-code-workspaces/<repo>/` to use

---

## Error Handling

- If `~/.claude-code-workspaces/` doesn't exist, create it on first use
- If repo subdirectory doesn't exist, create it on first use
- If a workspace name doesn't match any active workspace, check archived and suggest unarchive
- If git worktree commands fail, show the error and suggest troubleshooting steps
- Never lose `.workspace.json` metadata — it's the source of truth for branch tracking

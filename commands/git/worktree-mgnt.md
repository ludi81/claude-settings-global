---
argument-hint: [action] [args...]
description: Git worktree management - list, switch, open, or manage worktrees (generic, project-agnostic)
---

# Generic Git Worktree Management Command

You are helping manage git worktrees for any git project.

## User Request

$ARGUMENTS

## Configuration

This command uses an optional `.vscode/worktree-config.json` file in the project root to store shortcuts and preferences.

**IMPORTANT**: Before executing any commands, check if `.vscode/worktree-config.json` exists. If it does NOT exist:
1. Create the `.vscode/` directory if needed
2. Detect all existing worktrees using `git worktree list`
3. Auto-generate a config file with detected worktrees
4. Use the first worktree as "main", others as numbered shortcuts

### Config Schema

```json
{
  "shortcuts": {
    "main": "/absolute/path/to/main/worktree",
    "feature": "/absolute/path/to/feature/worktree",
    "review": "/absolute/path/to/review/worktree"
  },
  "packageManager": "npm",
  "defaultBranch": "main"
}
```

### Auto-Generation Logic

When generating the config:
1. Run `git worktree list` to detect all worktrees
2. Assign the first worktree (usually the main repo) to "main" shortcut
3. If there are additional worktrees, assign them to "feature", "review", etc.
4. Detect package manager by checking for lock files:
   - `pnpm-lock.yaml` → "pnpm"
   - `yarn.lock` → "yarn"
   - `package-lock.json` → "npm"
   - Default to "npm" if none found
5. Detect default branch by checking `git symbolic-ref refs/remotes/origin/HEAD` or default to "main"

## Available Actions

### Setup Commands
- **init** - Initialize worktree setup with configuration and optional worktree creation
  - Creates `.vscode/worktree-config.json` if it doesn't exist
  - Optionally creates feature and review worktrees in NoBackup folder
  - Asks user which worktrees to create (feature, review, or custom)
  - Detects package manager and default branch

### Information Commands
- **list** - List all worktrees with their branches and paths
- **status** - Show detailed status of all worktrees (branch, uncommitted changes, sync status)
- **info** - Show worktree configuration and shortcuts from config file

### Navigation Commands
- **goto <name>** - Print command to change directory to worktree (use shortcut name or path)
- **open <name>** - Open worktree in VS Code (use shortcut name or path)

### Management Commands
- **create <path> <branch>** - Create new worktree at path for branch
- **remove <path|name>** - Remove worktree (with confirmation)
- **switch <name> <branch>** - Switch branch in a worktree
- **sync** - Sync all worktrees (git fetch)
- **prune** - Clean up deleted worktrees

### Configuration Commands
- **config** - Show current configuration
- **config init** - Regenerate configuration file
- **config set <key> <value>** - Set a configuration value (e.g., `config set packageManager pnpm`)
- **config shortcut <name> <path>** - Add/update a shortcut

### Development Commands
- **install <name>** - Install dependencies in worktree (uses configured package manager)
- **dev <name> [port]** - Show command to start dev server in worktree
- **build <name>** - Run build in worktree
- **lint <name>** - Run linter in worktree
- **test <name>** - Run tests in worktree

## Implementation Guidelines

### 0. Init Command

**Purpose**: Set up worktree infrastructure for a project from scratch.

**Execution Flow**:

1. **Check current state**:
   ```bash
   # Get git root
   GIT_ROOT=$(git rev-parse --show-toplevel)

   # Check if config exists
   if [ -f "$GIT_ROOT/.vscode/worktree-config.json" ]; then
     echo "⚠️  Config already exists. Use 'config init' to regenerate."
     exit 1
   fi

   # Get current worktrees
   WORKTREES=$(git worktree list)
   WORKTREE_COUNT=$(echo "$WORKTREES" | wc -l)
   ```

2. **Detect environment**:
   ```bash
   # Detect package manager
   PM="npm"
   [ -f pnpm-lock.yaml ] && PM="pnpm"
   [ -f yarn.lock ] && PM="yarn"
   [ -f package-lock.json ] && PM="npm"

   # Detect default branch
   DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
   [ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH="main"

   # Get repo name
   REPO_NAME=$(basename "$GIT_ROOT")

   # Determine NoBackup folder location
   # Strategy: Check common locations, prefer /Users/aj/GIT/NoTMBackup
   if [ -d "/Users/aj/GIT/NoTMBackup" ]; then
     NOBACKUP_DIR="/Users/aj/GIT/NoTMBackup"
   elif [ -d "$GIT_ROOT/../NoBackup" ]; then
     NOBACKUP_DIR=$(cd "$GIT_ROOT/../NoBackup" && pwd)
   elif [ -d "$GIT_ROOT/../../NoBackup" ]; then
     NOBACKUP_DIR=$(cd "$GIT_ROOT/../../NoBackup" && pwd)
   else
     # Ask user for NoBackup location
     echo "NoBackup folder not found. Please provide path:"
     read -p "NoBackup directory: " NOBACKUP_DIR
   fi
   ```

3. **Ask user what to create**:
   - Show current state (how many worktrees exist)
   - Present options:
     * Create **feature** worktree (in `$NOBACKUP_DIR/$REPO_NAME-feature`)
     * Create **review** worktree (in `$NOBACKUP_DIR/$REPO_NAME-review`)
     * Create **custom** worktree (ask for name and path)
     * Skip worktree creation (config only)
   - Allow multiple selections

4. **Create worktrees** (if requested):
   ```bash
   # For each selected worktree type:
   # Example: feature worktree
   FEATURE_PATH="$NOBACKUP_DIR/$REPO_NAME-feature"

   # Ask for branch name
   echo "Branch name for feature worktree (leave empty to create new from $DEFAULT_BRANCH):"
   read -p "Branch: " FEATURE_BRANCH

   if [ -z "$FEATURE_BRANCH" ]; then
     # Create new branch from default
     FEATURE_BRANCH="feature/$(date +%Y%m%d)-work"
     git worktree add -b "$FEATURE_BRANCH" "$FEATURE_PATH" "$DEFAULT_BRANCH"
   else
     # Use existing branch
     git worktree add "$FEATURE_PATH" "$FEATURE_BRANCH"
   fi

   echo "✅ Created feature worktree at $FEATURE_PATH"
   ```

5. **Create configuration**:
   ```bash
   mkdir -p "$GIT_ROOT/.vscode"

   # Build shortcuts object
   cat > "$GIT_ROOT/.vscode/worktree-config.json" <<EOF
   {
     "shortcuts": {
       "main": "$GIT_ROOT",
       "feature": "$FEATURE_PATH",
       "review": "$REVIEW_PATH"
     },
     "packageManager": "$PM",
     "defaultBranch": "$DEFAULT_BRANCH"
   }
   EOF
   ```

6. **Summary**:
   - Show created worktrees
   - Show configuration file location
   - Suggest next steps (e.g., `install feature`, `open feature`)

**Interactive prompts** (use Claude's AskUserQuestion when appropriate):
- Which worktrees to create? (feature, review, both, none, custom)
- For each worktree: use existing branch or create new?
- If creating new: branch name suggestion (or auto-generate)
- NoBackup folder location (if not found)

**Example interaction**:
```
User: /xx_worktree-mgnt init

🔍 Current state:
- Repository: gbs-platform
- Git root: /Users/aj/Dropbox/GIT/GBS/gbs-platform
- Existing worktrees: 1 (main)
- Package manager: npm (detected from package-lock.json)
- Default branch: dev

📁 NoBackup folder: /Users/aj/GIT/NoTMBackup

Which worktrees would you like to create?
1. feature (gbs-platform-feature)
2. review (gbs-platform-review)
3. Custom (specify name)
4. None (config only)

Select options (comma-separated, e.g., 1,2): 1,2

Creating feature worktree...
Branch name (leave empty for auto-generated):
✅ Created branch: feature/20251022-work
✅ Created worktree at: /Users/aj/GIT/NoTMBackup/gbs-platform-feature

Creating review worktree...
Branch name (leave empty for auto-generated):
✅ Created branch: review/20251022
✅ Created worktree at: /Users/aj/GIT/NoTMBackup/gbs-platform-review

✅ Configuration saved to .vscode/worktree-config.json

Next steps:
- Install dependencies: /xx_worktree-mgnt install feature
- Open in VS Code: /xx_worktree-mgnt open feature
- List all worktrees: /xx_worktree-mgnt list
```

### 1. Configuration Management

**Before ANY command execution (except init):**
```bash
# Check if config exists
if [ ! -f .vscode/worktree-config.json ]; then
  # Suggest running init
  echo "⚠️  No configuration found. Run '/xx_worktree-mgnt init' first."
  exit 1
fi
```

**Load configuration** to resolve shortcuts to paths.

### 2. Information Commands

**list**: Use `git worktree list` and format output nicely with shortcuts from config

**status**: For each worktree:
- Show branch
- Run `git status --porcelain` to check uncommitted changes
- Run `git rev-list --left-right --count origin/$(branch)...HEAD` to check sync status

**info**: Display the contents of `.vscode/worktree-config.json` with formatting

**config**: Show current configuration with explanations

### 3. Navigation Commands

**goto**: Print the cd command
```
Run this command to change directory:
cd /path/to/worktree
```

**open**: Execute `code <path>` to open in VS Code

### 4. Management Commands

**create**:
```bash
git worktree add <path> <branch>
# Ask if dependencies should be installed
# Offer to add to config shortcuts
```

**remove**:
```bash
# Confirm first
git worktree remove <path>
# If fails, suggest: git worktree remove --force
# Update config to remove shortcut if exists
```

**switch**:
```bash
cd <worktree-path>
# Check for uncommitted changes
git status --porcelain
# If clean, switch
git checkout <branch>
# Check if package.json changed
git diff HEAD@{1} HEAD -- package.json package-lock.json
# If changed, suggest: <packageManager> ci
```

**sync**: Run `git fetch --all` from current directory

**prune**: Run `git worktree prune`

### 5. Configuration Commands

**config init**: Regenerate `.vscode/worktree-config.json` by detecting worktrees

**config set**: Update a top-level config value

**config shortcut**: Add or update a shortcut in the config

### 6. Development Commands

Use the configured package manager for all commands.

**install**:
```bash
cd <worktree-path>
<packageManager> ci
```

**dev/build/lint/test**: Show the command, don't run in background
```
Run this in a separate terminal:
cd /path/to/worktree && <packageManager> run <command>
```

## Shortcut Resolution

When a user provides a name argument (e.g., `open main`):
1. Check if it's a shortcut in the config file
2. If yes, use the configured path
3. If no, treat it as a path or worktree index number

Support these built-in shortcuts (if not defined in config):
- `main`, `m` → First worktree from `git worktree list`
- Index numbers → `1` for first worktree, `2` for second, etc.

## Package Manager Detection

If not in config, detect at runtime:
```bash
if [ -f pnpm-lock.yaml ]; then
  PM="pnpm"
elif [ -f yarn.lock ]; then
  PM="yarn"
elif [ -f package-lock.json ]; then
  PM="npm"
else
  PM="npm"
fi
```

## Important Notes

- Always work relative to git root (use `git rev-parse --show-toplevel`)
- Check if command is run inside a git repository
- Warn if trying to checkout same branch in multiple worktrees
- For new worktrees, suggest adding to config shortcuts
- Be concise - this is a management tool
- If config is outdated (references non-existent worktrees), warn user

## Example Workflows

### First-time use
```
User: /xx_worktree-mgnt list
→ Check if .vscode/worktree-config.json exists
→ If not, create it with detected worktrees
→ Show worktree list with assigned shortcuts
```

### Opening a worktree
```
User: /xx_worktree-mgnt open main
→ Load config
→ Resolve "main" to path from shortcuts
→ Execute: code /path/to/main
```

### Creating and adding to config
```
User: /xx_worktree-mgnt create ~/git/my-feature feat/new-feature
→ git worktree add ~/git/my-feature feat/new-feature
→ Ask: "Would you like to add this as a shortcut? (e.g., 'feature')"
→ If yes, update config
```

## Process the Request

1. **First**, check if `.vscode/worktree-config.json` exists
2. If not, auto-generate it (unless the command is a simple info command)
3. **Then**, process the user's arguments (`$ARGUMENTS`)
4. Be helpful and concise

Now execute the requested action.

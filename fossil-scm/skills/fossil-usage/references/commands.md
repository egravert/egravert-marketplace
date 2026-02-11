# Fossil SCM Complete Command Reference

Detailed command documentation for Fossil SCM 2.27. For quick reference and common workflows, see the main SKILL.md.

---

## Repository Setup

### Creating a New Repository

```bash
# Create a new empty repository
fossil init project.fossil

# Create without an initial empty commit (first commit becomes initial)
fossil init --empty project.fossil
```

### Cloning an Existing Repository

```bash
# Basic clone
fossil clone https://example.com/repo project.fossil

# Clone with username (will prompt for password)
fossil clone https://user@example.com/repo project.fossil

# Clone including private branches (requires 'x' capability)
fossil clone --private https://user@example.com/repo project.fossil

# Clone including unversioned files
fossil clone --unversioned https://example.com/repo project.fossil
```

### Opening a Repository (Creating a Working Checkout)

```bash
# Create working directory and open repository
mkdir project && cd project
fossil open ../project.fossil

# Open into existing non-empty directory (requires --force)
fossil open ../project.fossil --force

# Open a specific branch or version
fossil open ../project.fossil trunk
fossil open ../project.fossil v1.2.3

# Open with workdir option (creates checkout in specified dir)
fossil open project.fossil --workdir ./working
```

### Closing a Repository Connection

```bash
# Close the connection (keeps files, removes _FOSSIL_ link)
fossil close

# Force close even with uncommitted changes
fossil close --force
```

---

## Basic File Operations

### Adding Files

```bash
# Add specific files
fossil add file1.c file2.h

# Add all new files and remove deleted ones
fossil addremove

# Add files matching a pattern
fossil add src/*.c
```

### Removing Files

```bash
# Remove file from version control (keeps on disk)
fossil rm filename.txt

# Alternative syntax
fossil delete filename.txt

# Remove file from disk and version control
fossil rm --hard filename.txt
```

### Moving/Renaming Files

```bash
# Rename a file (records the rename for next commit)
fossil mv oldname.c newname.c

# Move file to different directory
fossil mv file.c src/file.c

# Note: This only records the rename in Fossil
# You must also rename the actual file on disk:
mv oldname.c newname.c
```

### Checking Status

```bash
# Show changed files since last commit
fossil changes

# Show full status including extras
fossil status

# List files not tracked by Fossil
fossil extras

# List all versioned files
fossil ls

# List files with detailed info
fossil ls -l
```

---

## Commits

### Basic Commit

```bash
# Commit all changes (opens editor for message)
fossil commit

# Commit with inline message
fossil commit -m "Fix bug in parser"

# Commit specific files only
fossil commit file1.c file2.c -m "Update these files"

# Shorthand alias
fossil ci -m "Commit message"
```

### Commit Options

```bash
# Create a new branch with this commit
fossil commit --branch feature-xyz -m "Start new feature"

# Add tags to the commit
fossil commit --tag release --tag v1.0.0 -m "Release version 1.0"

# Set background color for timeline display
fossil commit --bgcolor "#aaffaa" -m "Green commit"

# Allow commit even if it creates a fork
fossil commit --allow-fork -m "Intentional fork"

# Allow empty commit (no file changes)
fossil commit --allow-empty -m "Metadata only change"

# Private commit (never synced to remote)
fossil commit --private -m "Local experiment"

# Close the branch after this commit
fossil commit --close -m "Final commit on this branch"

# Specify date/time for historical imports
fossil commit --date-override "2024-01-15T10:30:00" -m "Historical commit"
```

### Amending Commits

The `amend` command modifies commit metadata without changing content:

```bash
# Change the commit message
fossil amend HASH -m "New commit message"

# Edit message in editor
fossil amend HASH --edit-comment

# Change the author
fossil amend HASH --author "Jane Doe"

# Move commit to a different branch
fossil amend HASH --branch new-branch-name

# Add a tag to an existing commit
fossil amend HASH --tag release-1.0

# Remove/cancel a tag
fossil amend HASH --cancel old-tag

# Close a leaf (mark branch as finished)
fossil amend HASH --close

# Change background color
fossil amend HASH --bgcolor "#ffcccc"

# Hide a branch from default timeline views
fossil amend HASH --hide

# Dry run (show what would happen)
fossil amend HASH --dry-run -m "Test message"
```

---

## Branching

### Branch Concepts

In Fossil, branches are identified by tags on commits. The "trunk" branch is the default mainline. Unlike Git, Fossil encourages creating branches at the point of need during commit, rather than in advance.

### Creating Branches

```bash
# Preferred: Create branch during commit
fossil commit --branch feature-xyz -m "Start feature XYZ"

# Create branch in advance (discouraged but supported)
fossil branch new feature-xyz trunk

# Create branch with custom background color
fossil branch new bugfix-123 trunk --bgcolor "#ffaaaa"

# Create private branch (not synced)
fossil branch new experiment current --private
```

### Listing Branches

```bash
# List open branches (default)
fossil branch list

# List all branches including closed
fossil branch list --all

# List only closed branches
fossil branch list --closed

# List branches merged into current
fossil branch list --merged

# List branches NOT merged into current
fossil branch list --unmerged

# Show recently changed branches
fossil branch lsh
fossil branch lsh 10  # Show last 10

# List private branches only
fossil branch list -p

# Show branches matching pattern
fossil branch list "feature-*"

# Show current branch name
fossil branch current
```

### Switching Branches

```bash
# Switch to a branch (with autosync)
fossil update trunk
fossil update feature-xyz

# Switch to specific commit
fossil update abc123def

# Switch without autosync, overwrite local changes
fossil checkout trunk --force

# Switch to the latest commit on current branch
fossil update
fossil update latest
```

### Closing and Reopening Branches

```bash
# Close a branch (mark as finished)
fossil branch close feature-xyz

# Close multiple branches at once
fossil branch close branch1 branch2 branch3

# Reopen a closed branch
fossil branch reopen feature-xyz

# Alternative: Use amend to close
fossil amend HASH --close

# Hide a branch from timeline (but keep open)
fossil branch hide old-experiment

# Unhide a branch
fossil branch unhide old-experiment
```

### Branch Information

```bash
# Show info about a specific branch
fossil branch info feature-xyz
```

---

## Merging

### Basic Merge

```bash
# Update to target branch first
fossil update trunk

# Merge source branch into current checkout
fossil merge feature-xyz

# Test the merge, then commit
fossil commit -m "Merged feature-xyz into trunk"
```

### Merge Options

```bash
# Merge and close the source branch on commit
fossil merge --integrate feature-xyz
fossil commit -m "Integrated feature-xyz"

# Cherry-pick a single commit
fossil merge --cherrypick abc123def

# Back out (reverse) a specific commit
fossil merge --backout abc123def

# Merge using specific baseline
fossil merge --baseline v1.0 feature-branch

# Force merge even with conflicts
fossil merge --force feature-xyz

# Dry run (show what would be merged)
fossil merge --dry-run feature-xyz
```

### Handling Merge Conflicts

When conflicts occur, Fossil marks them in files with conflict markers:

```bash
# After merge with conflicts, files contain:
# >>>>>>> BEGIN MERGE CONFLICT
# ... your version ...
# ======= COMMON ANCESTOR
# ... common ancestor ...
# ||||||| ORIGINAL
# ... their version ...
# <<<<<<< END MERGE CONFLICT

# Edit files to resolve conflicts, then:
fossil commit -m "Resolved merge conflicts"

# Or abort the merge
fossil revert
fossil undo  # Alternative
```

### Resolving Forks

```bash
# If you accidentally create a fork, merge without arguments
# finds and merges the fork automatically
fossil merge
```

---

## Synchronization (Multi-User Workflows)

### Autosync Mode (Default)

Autosync automatically pushes after commit and pulls before update:

```bash
# Check current autosync setting
fossil settings autosync

# Enable autosync (default)
fossil settings autosync on

# Disable autosync for manual control
fossil settings autosync off

# Autosync only on pull (before commit)
fossil settings autosync pullonly
```

### Manual Sync Operations

```bash
# Full sync (push and pull)
fossil sync

# Sync with specific remote
fossil sync https://example.com/repo

# Push local changes to remote
fossil push

# Pull remote changes to local
fossil pull

# Sync including private branches
fossil sync --private

# Sync unversioned files
fossil sync --unversioned
```

### Remote Management

```bash
# Show current remote URL
fossil remote

# Set default remote
fossil remote add https://example.com/repo

# List configured remotes
fossil remote list

# Remove a remote
fossil remote delete origin
```

### Configuration Sync

```bash
# Pull all configuration from remote (requires Setup capability)
fossil configuration pull all

# Push configuration to remote
fossil configuration push all

# Sync specific config areas
fossil configuration pull skin
fossil configuration pull shun
fossil configuration pull user
```

---

## The Stash

The stash temporarily saves uncommitted changes:

### Basic Stash Operations

```bash
# Save current changes to stash and revert working dir
fossil stash save -m "Work in progress"

# Save without message (will prompt)
fossil stash save

# Save but keep working directory unchanged
fossil stash snapshot -m "Checkpoint"

# Stash specific files only
fossil stash save file1.c file2.c -m "Partial stash"
```

### Listing and Viewing Stashes

```bash
# List all stashes
fossil stash list

# List with verbose file details
fossil stash list -v

# Show contents of most recent stash as diff
fossil stash show

# Show specific stash
fossil stash show 1

# Show using graphical diff tool
fossil stash gshow
```

### Applying Stashes

```bash
# Apply most recent stash and remove it
fossil stash pop

# Apply specific stash and remove it
fossil stash pop 2

# Apply stash but keep it in stash list
fossil stash apply

# Apply specific stash
fossil stash apply 3

# Apply stash to its original baseline
fossil stash goto 2
```

### Managing Stashes

```bash
# Delete a specific stash
fossil stash drop 1

# Delete multiple stashes
fossil stash rm 1 2 3

# Delete all stashes (not undoable!)
fossil stash drop --all

# Rename a stash
fossil stash rename 1 "Better description"

# Compare stash with current working directory
fossil stash diff 1
fossil stash gdiff 1  # Graphical diff
```

---

## Tags

Tags are labels attached to commits for easy reference:

### Adding Tags

```bash
# Add tag during commit
fossil commit --tag v1.0.0 -m "Release 1.0.0"

# Add tag to existing commit
fossil tag add v1.0.0 abc123def

# Add tag with current (tip) commit
fossil tag add v1.0.0 current

# Add propagating tag (applies to all descendants)
fossil tag add --propagate release-branch abc123def

# Add raw tag (used internally, e.g., for closing)
fossil tag add --raw closed abc123def
```

### Listing and Finding Tags

```bash
# List all tags
fossil tag list

# Find commits with specific tag
fossil tag find v1.0.0

# Show tags on current checkout
fossil info
```

### Removing Tags

```bash
# Cancel/remove a tag
fossil tag cancel v1.0.0 abc123def

# Cancel raw tag
fossil tag cancel --raw closed abc123def
```

---

## Ticketing System

Fossil includes a built-in issue tracker. The ticket system is highly customizable, but the CLI commands work with whatever fields are configured.

### Listing Tickets

```bash
# List tickets using a named report
fossil ticket show "All Tickets"

# Use report number (0 = all columns)
fossil ticket show 0

# Apply filter to limit results
fossil ticket show "Open Tickets" "status='Open'"

# Change output delimiter (default is TAB)
fossil ticket show "All Tickets" --limit ","

# Quote special characters in output
fossil ticket show "All Tickets" --quote
```

### Listing Available Fields and Reports

```bash
# List all ticket fields defined in the repository
fossil ticket list fields
fossil ticket ls fields

# List all configured ticket reports
fossil ticket list reports
fossil ticket ls reports
```

### Creating Tickets

```bash
# Add a new ticket with field values
fossil ticket add title "Bug description" status "Open" priority "High"

# Add ticket with multiple fields
fossil ticket add \
  title "Login fails on Safari" \
  type "Bug" \
  status "Open" \
  priority "Critical" \
  subsystem "Authentication"

# Add ticket with multiline text (use --quote)
fossil ticket add --quote \
  title "Complex issue" \
  comment "Line 1\nLine 2\nLine 3"
```

### Modifying Tickets

```bash
# Change a ticket field (requires ticket UUID)
fossil ticket set abc123 status "Closed"

# Alternative syntax
fossil ticket change abc123 status "Closed"

# Change multiple fields at once
fossil ticket set abc123 \
  status "Fixed" \
  resolution "Code change" \
  priority "Low"

# Append to a field (use +FIELD)
fossil ticket set abc123 +comment "Additional note added"

# Set multiline content with --quote
fossil ticket set abc123 --quote \
  comment "First line\nSecond line"
```

### Viewing Ticket History

```bash
# Show complete change history for a ticket
fossil ticket history abc123def
```

### Common Ticket Fields

Default ticket fields (can be customized per repository):

| Field | Description |
|-------|-------------|
| `title` | Short description of the issue |
| `type` | Bug, Feature Request, Documentation, etc. |
| `status` | Open, Closed, Fixed, Review, etc. |
| `priority` | Critical, High, Medium, Low |
| `severity` | Cosmetic, Minor, Major, Critical |
| `subsystem` | Component or module affected |
| `resolution` | How the issue was resolved |
| `foundin` | Version where issue was found |
| `comment` | Discussion/notes (often appendable) |
| `private_contact` | Reporter's email (not public) |

### Ticket UUID

Tickets are identified by their artifact UUID. Use a shortened prefix if unambiguous:

```bash
# Full UUID
fossil ticket set f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5 status "Closed"

# Shortened (if unambiguous)
fossil ticket set f4a3b2 status "Closed"
```

### Tips for CLI Ticket Management

1. **Find ticket UUIDs**: Use `fossil ticket show 0` to see all tickets with their UUIDs
2. **Custom reports**: Create reports in the web UI, then use them from CLI
3. **Scripting**: Use `--quote` for predictable output parsing
4. **Field names**: Use `fossil ticket ls fields` to see exact field names
5. **Validation**: Field values are NOT validated—ensure correct values

---

## Diff and History

### Viewing Differences

```bash
# Diff working directory vs last commit
fossil diff

# Diff specific file
fossil diff file.c

# Diff between two versions
fossil diff --from abc123 --to def456

# Graphical diff (uses gdiff-command setting)
fossil gdiff

# Side-by-side diff
fossil diff --side-by-side

# Unified diff with context
fossil diff -c 5  # 5 lines of context

# Show diff statistics (lines added/removed)
fossil diff --numstat
```

### File History

```bash
# Show history of a file
fossil finfo filename.c

# Show history with patches
fossil finfo -p filename.c

# Show file at specific version
fossil cat filename.c -r abc123

# Annotate file (show who changed each line)
fossil annotate filename.c

# Blame (alias for annotate)
fossil blame filename.c
```

### Timeline

```bash
# Show recent timeline
fossil timeline

# Limit number of entries
fossil timeline -n 20

# Show specific types only
fossil timeline -t ci    # Check-ins only
fossil timeline -t w     # Wiki changes
fossil timeline -t t     # Ticket changes

# Timeline for specific branch
fossil timeline -b trunk

# Timeline for specific path
fossil timeline -p src/

# Format timeline output
fossil timeline --format "%h %d %c"
```

---

## Undo and Revert

### Undoing Operations

```bash
# Undo last update, merge, or revert
fossil undo

# Redo what was undone
fossil redo

# Undo changes to specific files
fossil undo file1.c file2.c
```

### Reverting Changes

```bash
# Revert all uncommitted changes
fossil revert

# Revert specific files
fossil revert file1.c file2.c

# Revert to specific version
fossil revert -r abc123 file.c
```

### Cleaning Extra Files

```bash
# List extra (untracked) files
fossil extras

# Delete all extra files (interactive)
fossil clean

# Force delete without prompts
fossil clean --force

# Include dotfiles
fossil clean --dotfiles

# Dry run (show what would be deleted)
fossil clean --dry-run
```

---

## Settings

### Viewing and Changing Settings

```bash
# List all settings and their values
fossil settings

# View specific setting
fossil settings autosync

# Set a local (repository-specific) setting
fossil settings autosync off

# Set a global setting (all repositories)
fossil settings autosync off --global

# Unset a local setting (reverts to global)
fossil unset autosync
```

### Important Settings

| Setting | Description |
|---------|-------------|
| `autosync` | Auto push/pull on commit/update |
| `editor` | Text editor for commit messages |
| `gdiff-command` | External graphical diff tool |
| `proxy` | HTTP proxy URL |
| `case-sensitive` | Case sensitivity for filenames |
| `crlf-glob` | Files allowed to have CRLF line endings |
| `ignore-glob` | Patterns for files to ignore |
| `binary-glob` | Patterns for binary files |
| `clean-glob` | Files removed by `fossil clean` |
| `keep-glob` | Files NOT removed by `fossil clean` |

---

## Web Interface

### Starting the Web UI

```bash
# Open web UI in browser (local only)
fossil ui

# Start server on specific port
fossil ui --port 9000

# Start as server (external access)
fossil server --port 8080

# Serve multiple repositories in a directory
fossil server /path/to/repos --repolist
```

---

## Useful Patterns

### Starting a New Project

```bash
fossil init myproject.fossil
mkdir myproject && cd myproject
fossil open ../myproject.fossil
fossil add .
fossil commit -m "Initial commit"
```

### Feature Branch Workflow

```bash
# Start a feature
fossil commit --branch feature-login -m "Start login feature"

# Work on the feature
fossil commit -m "Add login form"
fossil commit -m "Add validation"

# Merge back to trunk
fossil update trunk
fossil merge --integrate feature-login
fossil commit -m "Merged feature-login"
```

### Collaborative Workflow

```bash
# Alice: Clone and work
fossil clone https://server/repo alice.fossil
mkdir work && cd work
fossil open ../alice.fossil
fossil commit -m "Alice's changes"  # Autosync pushes

# Bob: His changes auto-arrive on update
fossil update  # Pulls Alice's changes automatically
```

### Handling Mistakes

```bash
# Wrong commit? Move to "mistake" branch
fossil amend HEAD --branch mistake
fossil amend HEAD --close
fossil update trunk

# Accidentally committed sensitive data?
# Contact repository admin for shunning
fossil shun ARTIFACT-ID  # Requires admin access
```

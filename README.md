# jj-stack: Jujutsu Stack PR Management Tool

A command-line utility to automate pushing stacks of Jujutsu (JJ) changes to GitHub as pull requests. The tool manages bookmark creation, PR relationships, cross-references, and handles stack updates over time.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Commands](#commands)
- [Workflows](#workflows)
- [How It Works](#how-it-works)
- [Troubleshooting](#troubleshooting)
- [Examples](#examples)

## Overview

`jj-stack` bridges the gap between Jujutsu's powerful change-based workflow and GitHub's pull request system. It allows you to work with stacks of changes in JJ and automatically manage them as properly linked pull requests on GitHub.

### Key Concepts

- **Stack**: A sequence of JJ changes built on top of each other (e.g., trunk → change A → change B → change C)
- **Bookmark**: A Git branch that points to a specific JJ change
- **Stacked PRs**: Pull requests where each PR is based on the previous PR's branch, not on master

## Features

- ✅ **Automatic Bookmark Creation**: Generates descriptive bookmarks for each change
- ✅ **Proper PR Stacking**: Each PR is based on the previous PR in the stack
- ✅ **Cross-References**: PRs include links to parent and child PRs
- ✅ **Update Detection**: Detects changes and updates existing PRs (no duplicates)
- ✅ **Stateless Operation**: Queries JJ and GitHub each time (no state files)
- ✅ **Cleanup**: Removes bookmarks for merged/closed PRs
- ✅ **Multi-User Support**: Bookmark names include GitHub username

## Prerequisites

1. **Jujutsu (jj)**: Version control system
   ```bash
   # Install via Homebrew (macOS)
   brew install jj

   # Or follow instructions at: https://github.com/martinvonz/jj
   ```

2. **GitHub CLI (gh)**: GitHub command-line tool
   ```bash
   # Install via Homebrew
   brew install gh

   # Authenticate with GitHub
   gh auth login
   ```

3. **JJ Repository Configuration**: Define `trunk()` in your repo
   ```bash
   # Edit .jj/repo/config.toml and add:
   [revset-aliases]
   'trunk()' = 'master@origin'
   ```

## Installation

The tool is already installed at `~/bin/jj-stack`. Ensure `~/bin` is in your PATH:

```bash
# Add to ~/.zshrc or ~/.bashrc if needed
export PATH="$HOME/bin:$PATH"

# Verify installation
jj-stack --help
```

## Quick Start

```bash
# 1. Create a stack of changes in JJ
jj commit -m "Add feature foundation"
jj commit -m "Build on foundation"
jj commit -m "Add documentation"

# 2. Push stack to GitHub as PRs
jj-stack push

# 3. Check status
jj-stack status

# 4. After merging PRs, clean up bookmarks
jj-stack cleanup
```

## Commands

### `jj-stack push`

Push the current stack to GitHub, creating or updating pull requests.

**Behavior:**
- Creates bookmarks for changes without bookmarks
- Pushes bookmarks to GitHub
- Creates new PRs or updates existing ones
- Sets correct base branches (stacked on each other)
- Updates PR descriptions with cross-references

**Usage:**
```bash
cd ~/my-jj-repo
jj-stack push
```

**Output Example:**
```
Found 3 change(s) in stack

Creating bookmarks and pushing to GitHub...
  spzvyltwqokl: jingibus/2026-02-18/add-feature-foundation-spzvyltwqokl
  nxtzpqwnpqvy: jingibus/2026-02-18/build-on-foundation-nxtzpqwnpqvy
  tqruqxprvowv: jingibus/2026-02-18/add-documentation-tqruqxprvowv

Creating/updating PRs...
  ✓ Created PR #123: Add feature foundation
  ✓ Created PR #124: Build on foundation
  ✓ Created PR #125: Add documentation

Stack pushed successfully! (3 changes)
```

### `jj-stack status`

Show the status of the current stack and associated PRs.

**Usage:**
```bash
jj-stack status
```

**Output Example:**
```
Current Stack Status (GitHub user: jingibus)
============================================================

1. Add feature foundation
   Change ID: spzvyltwqokl
   Bookmark: jingibus/2026-02-18/add-feature-foundation-spzvyltwqokl
   PR: #123 (OPEN)
   URL: https://github.com/owner/repo/pull/123

2. Build on foundation
   Change ID: nxtzpqwnpqvy
   Bookmark: jingibus/2026-02-18/build-on-foundation-nxtzpqwnpqvy
   PR: #124 (OPEN)
   URL: https://github.com/owner/repo/pull/124

3. Add documentation
   Change ID: tqruqxprvowv
   Bookmark: jingibus/2026-02-18/add-documentation-tqruqxprvowv
   PR: #125 (OPEN)
   URL: https://github.com/owner/repo/pull/125

============================================================
Total: 3 change(s)
```

### `jj-stack list`

List all changes in the current stack with metadata (no GitHub queries).

**Usage:**
```bash
jj-stack list
```

**Output Example:**
```
Stack Changes (trunk()..@):
============================================================

1. Add feature foundation
   Change ID: spzvyltwqokl
   Commit ID: 97b06d25c67d
   Parent: trunk

2. Build on foundation
   Change ID: nxtzpqwnpqvy
   Commit ID: 5b82f1a2efab
   Parent: spzvyltwqokl

3. Add documentation
   Change ID: tqruqxprvowv
   Commit ID: ed8a1fa940c1
   Parent: nxtzpqwnpqvy

============================================================
Total: 3 change(s)
```

### `jj-stack sync`

Fetch latest trunk, rebase the stack, and update all PRs.

**Behavior:**
1. Runs `jj git fetch` to get latest trunk
2. Rebases stack with `jj rebase -d trunk()`
3. Updates bookmarks to point to rebased commits
4. Force-pushes updated bookmarks
5. Updates PR descriptions and base branches

**Usage:**
```bash
jj-stack sync
```

**Output Example:**
```
Fetching latest trunk...
Rebasing stack...
Updating bookmarks...
Force-pushing updated bookmarks...
Updating PRs...
  ✓ Updated PR #123
  ✓ Updated PR #124
  ✓ Updated PR #125

Stack synced successfully!
```

### `jj-stack cleanup`

Remove bookmarks for merged or closed PRs.

**Behavior:**
- Finds all bookmarks for your GitHub username
- Checks PR status for each bookmark
- Deletes bookmarks for merged/closed PRs (locally and remotely)

**Usage:**
```bash
jj-stack cleanup
```

**Output Example:**
```
Found 5 bookmark(s) for jingibus
Cleaning up bookmark jingibus/2026-02-15/old-feature-abc123 (PR #100 is MERGED)
Cleaning up bookmark jingibus/2026-02-16/closed-feature-def456 (PR #110 is CLOSED)
Pushing bookmark deletions to remote...

Cleaned up 2 bookmark(s)
```

## Workflows

### Basic Workflow: Create and Push a Stack

```bash
# 1. Start from trunk
jj new trunk()

# 2. Create first change
echo "feature code" > feature.rs
jj commit -m "Add feature foundation"

# 3. Create second change
echo "more code" >> feature.rs
jj commit -m "Extend feature"

# 4. Create third change
echo "# Docs" > FEATURE.md
jj commit -m "Document feature"

# 5. Push entire stack
jj-stack push

# Result: 3 PRs created, stacked on each other
```

### Updating a Change in the Stack

```bash
# 1. Edit a change in the middle of the stack
jj edit <change-id-2>

# 2. Make your changes
echo "updated code" >> feature.rs

# 3. Amend the change
jj commit --amend -m "Extend feature (updated)"

# 4. Return to top of stack
jj edit @

# 5. Push updates (updates PR #2 and rebases PR #3)
jj-stack push
```

### Syncing with Latest Trunk

```bash
# Fetch trunk, rebase stack, update all PRs
jj-stack sync

# If conflicts occur:
# 1. Resolve conflicts manually
# 2. Run: jj-stack sync again
```

### Merging PRs and Cleaning Up

```bash
# 1. Merge bottom PR on GitHub (PR #1)
gh pr merge 1 --squash

# 2. Clean up bookmark
jj-stack cleanup

# 3. Update remaining PRs to rebase on trunk
# (You may need to manually update base branches via GitHub UI
#  or by editing and re-pushing the stack)
```

### Working with Multiple Stacks

```bash
# Stack 1: Feature A
jj new trunk()
jj commit -m "Feature A part 1"
jj commit -m "Feature A part 2"
jj-stack push

# Stack 2: Feature B (independent)
jj new trunk()
jj commit -m "Feature B part 1"
jj commit -m "Feature B part 2"
jj-stack push

# Each stack gets its own PRs with unique bookmarks
```

## How It Works

### Bookmark Naming Convention

Bookmarks follow the pattern:
```
<github-username>/<date>/<slug>-<change_id_short>
```

**Example:**
```
jingibus/2026-02-18/add-authentication-feature-spzvyltwqokl
```

**Components:**
- `jingibus`: GitHub username (from `gh` CLI)
- `2026-02-18`: Date when bookmark was created
- `add-authentication-feature`: Human-readable slug from commit message
- `spzvyltwqokl`: Short hash of JJ change ID (uniqueness)

### Change-to-PR Mapping

The tool is **stateless** and determines which PR belongs to which change by:

1. Querying GitHub for all open PRs
2. Checking if PR head branch name ends with `-<change_id>`
3. Matching change IDs from bookmark names to PR branch names

**Example:**
- Change ID: `spzvyltwqokl`
- Bookmark: `jingibus/2026-02-18/add-feature-spzvyltwqokl`
- GitHub searches for PR with head branch ending in `-spzvyltwqokl`

### PR Cross-References

Each PR description includes:

```markdown
<original commit message>

---

**Stack Information:**
- **Stack Position:** 2 of 3
- **Parent PR:** #123 (Add feature foundation)
- **Child PRs:** #125

**JJ Change ID:** `nxtzpqwnpqvy`

---
_This PR is part of a stack managed by jj-stack._
```

### Stack Determination

The tool uses the JJ revset: `trunk()..@`

This returns all changes between trunk (typically `master@origin`) and your current working copy (`@`), in topological order.

## Troubleshooting

### Error: "jj is not available"

**Solution:** Install Jujutsu:
```bash
brew install jj
```

### Error: "gh is not available" or "not authenticated"

**Solution:** Install and authenticate GitHub CLI:
```bash
brew install gh
gh auth login
```

### Error: "Not in a Jujutsu repository"

**Solution:** Navigate to a JJ repository or initialize one:
```bash
cd ~/my-project
jj init --git-repo .
```

### Error: "Failed to resolve trunk()"

**Solution:** Define `trunk()` in your repo config:
```bash
# Edit .jj/repo/config.toml
[revset-aliases]
'trunk()' = 'master@origin'
```

Or use a different branch:
```bash
'trunk()' = 'main@origin'
```

### Warning: "Failed to push bookmark"

**Cause:** Bookmark isn't tracked with remote.

**Solution:** The tool automatically tracks new bookmarks. If you see this error, try:
```bash
jj bookmark track <bookmark-name> --remote=origin
jj git push --bookmark <bookmark-name>
```

### PR Base Branch is Wrong

**Solution:** Run `jj-stack push` again to update PR base branches:
```bash
jj-stack push
```

The tool automatically detects correct base branches and updates them.

### Bookmarks Not Cleaned Up

**Solution:** Ensure cleanup is run after PRs are merged:
```bash
# Check PR status
gh pr list --state merged

# Run cleanup
jj-stack cleanup
```

## Examples

### Example 1: Simple 3-Change Stack

```bash
# Create changes
$ jj commit -m "Add user model"
$ jj commit -m "Add user service"
$ jj commit -m "Add user API endpoint"

# Push stack
$ jj-stack push
Found 3 change(s) in stack
Creating bookmarks and pushing to GitHub...
Creating/updating PRs...
  ✓ Created PR #201: Add user model
  ✓ Created PR #202: Add user service
  ✓ Created PR #203: Add user API endpoint
Stack pushed successfully! (3 changes)

# Result on GitHub:
# - PR #201: master → add-user-model
# - PR #202: add-user-model → add-user-service
# - PR #203: add-user-service → add-user-api-endpoint
```

### Example 2: Update Middle Change

```bash
# Edit middle change
$ jj edit <change-id-2>
$ echo "improved code" >> service.rs
$ jj commit --amend -m "Add improved user service"

# Return to top
$ jj edit @

# Push updates
$ jj-stack push
Found 3 change(s) in stack
Creating bookmarks and pushing to GitHub...
Creating/updating PRs...
  ✓ Updated PR #201: Add user model
  ✓ Updated PR #202: Add improved user service
  ✓ Updated PR #203: Add user API endpoint
Stack pushed successfully! (3 changes)

# PR #202 is updated with new code and title
# PR #203 is automatically rebased on updated PR #202
```

### Example 3: Merge and Clean Up

```bash
# Merge bottom PR
$ gh pr merge 201 --squash
✓ Merged #201

# Clean up bookmark
$ jj-stack cleanup
Found 3 bookmark(s) for jingibus
Cleaning up bookmark jingibus/2026-02-18/add-user-model-abc123 (PR #201 is MERGED)
Pushing bookmark deletions to remote...
Cleaned up 1 bookmark(s)

# Verify
$ jj bookmark list
jingibus/2026-02-18/add-user-service-def456: ...
jingibus/2026-02-18/add-user-api-endpoint-ghi789: ...
master: ...
```

## Architecture Notes

### Design Principles

1. **Stateless**: No persistent state files. Everything is derived from JJ and GitHub on each run.
2. **Bookmark-Based**: Uses bookmark names to map changes to PRs.
3. **Idempotent**: Running `push` multiple times doesn't create duplicates.
4. **User-Scoped**: Bookmark names include GitHub username to support multiple developers.

### Why Stateless?

- No state synchronization issues
- No corruption or stale state
- Works correctly after force-pushes, rebases, etc.
- Simpler to reason about and debug

### Limitations

- Requires unique JJ change IDs (always true in JJ)
- Assumes one PR per change (by design)
- Bookmark names include date (helps organize, but changes daily)

## Contributing

This tool is located at `~/bin/jj-stack` and is written in Python 3.

To modify:
```bash
# Edit the script
vim ~/bin/jj-stack

# Test changes
cd ~/Development/jj-test
jj-stack <command>
```

## License

This tool is provided as-is for personal use.

## Support

For issues or questions, refer to:
- [Jujutsu Documentation](https://github.com/martinvonz/jj/blob/main/docs/README.md)
- [GitHub CLI Documentation](https://cli.github.com/manual/)

---

**Version:** 1.0.0
**Last Updated:** 2026-02-18
**Author:** Built with Claude Code

# jj-stack Quick Reference

## Commands

| Command | Description |
|---------|-------------|
| `jj-stack push` | Push stack to GitHub, create/update PRs |
| `jj-stack status` | Show stack status with PR info |
| `jj-stack list` | List changes in stack (no GitHub queries) |
| `jj-stack sync` | Fetch trunk, rebase, update PRs |
| `jj-stack cleanup` | Remove bookmarks for merged/closed PRs |

## Common Workflows

### Create and Push Stack
```bash
jj commit -m "Change 1"
jj commit -m "Change 2"
jj commit -m "Change 3"
jj-stack push
```

### Update a Change
```bash
jj edit <change-id>
# make changes
jj commit --amend -m "Updated message"
jj edit @
jj-stack push
```

### Check Status
```bash
jj-stack status
```

### Sync with Trunk
```bash
jj-stack sync
```

### After Merging PRs
```bash
gh pr merge <pr-number> --squash
jj-stack cleanup
```

## Bookmark Format

```
<username>/<date>/<slug>-<change_id>
```

Example: `jingibus/2026-02-18/add-feature-spzvyltwqokl`

## Prerequisites

```bash
# Install tools
brew install jj gh

# Authenticate GitHub
gh auth login

# Configure trunk in .jj/repo/config.toml
[revset-aliases]
'trunk()' = 'master@origin'
```

## PR Cross-References

Each PR includes:
- Stack position (e.g., "2 of 3")
- Parent PR link
- Child PR links
- JJ change ID

## Tips

- **Run from repo root**: `jj-stack` works in any JJ repo
- **Idempotent**: Safe to run `push` multiple times
- **Stateless**: No state files to manage
- **Multi-user**: Bookmark names include GitHub username

## Troubleshooting

| Error | Solution |
|-------|----------|
| "jj is not available" | `brew install jj` |
| "gh is not available" | `brew install gh && gh auth login` |
| "Not in a JJ repository" | `cd` to JJ repo |
| "Failed to resolve trunk()" | Define `trunk()` in `.jj/repo/config.toml` |

## Getting Help

```bash
jj-stack --help
```

Full documentation: `~/bin/jj-stack-README.md`

# jj-stack Architecture and Design

## Overview

`jj-stack` is a Python 3 command-line tool that bridges Jujutsu's change-based workflow with GitHub's pull request system. This document explains the architectural decisions, implementation details, and design rationale.

## Design Principles

### 1. Stateless Operation

**Decision:** No persistent state storage.

**Rationale:**
- Eliminates state synchronization issues
- Cannot become stale or corrupted
- Resilient to force-pushes, rebases, and other disruptive operations
- Simpler mental model for users
- Easier to debug and maintain

**Implementation:**
- Every command queries JJ and GitHub fresh
- Change-to-PR mapping is derived from bookmark names
- No `.jj-stack` state directory or files

### 2. Bookmark-Based Identification

**Decision:** Use bookmark names to identify which PR belongs to which change.

**Rationale:**
- Bookmarks (Git branches) are the natural bridge between JJ and Git/GitHub
- Change ID embedded in bookmark name provides reliable mapping
- GitHub username prefix enables multi-user collaboration
- Human-readable slugs improve UX

**Bookmark Format:**
```
<github-username>/<date>/<slug>-<change_id_short>
```

**Example:**
```
jingibus/2026-02-18/add-authentication-spzvyltwqokl
```

**Components:**
- `github-username`: From `gh api user -q .login`
- `date`: `YYYY-MM-DD` format for temporal grouping
- `slug`: First 5 words from commit message, sanitized
- `change_id_short`: JJ's short change ID (12 chars)

### 3. Two-Pass Push Strategy

**Decision:** Separate bookmark creation/push from PR creation/update.

**Rationale:**
- Ensures all bookmarks exist before creating PRs
- Allows PRs to reference each other's branches as base
- Simplifies error handling (can retry PR creation without re-pushing)

**Implementation:**
```python
# Pass 1: Create and push bookmarks
for change in changes:
    bookmark = ensure_bookmark(change)
    git_push(bookmark)

# Pass 2: Create or update PRs
for change in changes:
    if pr_exists:
        update_pr()
    else:
        create_pr()
```

### 4. Idempotent Operations

**Decision:** Running `push` multiple times should be safe and not create duplicates.

**Rationale:**
- Users may run command accidentally or multiple times
- Network failures may require retries
- Should be safe to use in scripts

**Implementation:**
- Check if PR already exists before creating
- Use `find_pr_for_change()` to query GitHub by bookmark name
- Update existing PRs instead of creating new ones

## Architecture Components

### Core Data Model

```python
@dataclass
class Change:
    """Represents a JJ change."""
    change_id: str      # spzvyltwqokl
    commit_id: str      # 97b06d25c67d
    description: str    # Multi-line commit message
```

### Command Flow

```
User Command
    ↓
main()
    ↓
check_prerequisites()  # Verify jj, gh, auth
    ↓
Command Handler (cmd_push, cmd_status, etc.)
    ↓
JJ Queries / GitHub API Calls
    ↓
Output to User
```

### Key Algorithms

#### Stack Querying

```python
def get_stack_changes() -> List[Change]:
    """Query JJ for all changes in current stack."""
    # Uses revset: trunk()..@
    # Returns changes in topological order
```

**Revset:** `trunk()..@`
- `trunk()`: User-defined alias (typically `master@origin`)
- `@`: Current working copy
- `..`: All changes between (exclusive..inclusive)
- `--reversed`: Output in topological order (bottom to top)

**Template:**
```
change_id.short() ++ "\n" ++ commit_id.short() ++ "\n" ++ description ++ "\x00"
```

**Parsing:**
- Split by `\x00` (null byte) to separate changes
- Split each change by `\n` to get fields
- First line: change ID
- Second line: commit ID
- Rest: description

#### Change-to-PR Mapping

```python
def find_pr_for_change(change_id: str, username: str, state: str = 'open') -> Optional[int]:
    """Find PR by querying GitHub and matching bookmark pattern."""
    prs = gh_pr_list(state=state)
    for pr in prs:
        if pr.head_ref.startswith(f"{username}/") and pr.head_ref.endswith(f"-{change_id}"):
            return pr.number
    return None
```

**How it works:**
1. Get all PRs in specified state from GitHub
2. Filter by username prefix in head branch
3. Match by change ID suffix in head branch
4. Return PR number if found

**Why this works:**
- Bookmark names are unique (include change ID)
- Change IDs are stable in JJ (don't change with rebases)
- GitHub stores branch name in PR metadata

#### Parent Detection

```python
def get_parent_change_id(change_id: str) -> Optional[str]:
    """Get parent change ID using JJ's revset syntax."""
    # Query: <change_id>-
    # Returns: parent of change_id
    parent = jj_log(f"{change_id}-")
    if parent == trunk_id:
        return None  # Parent is trunk
    return parent
```

**Revset:** `<change_id>-`
- `-` suffix means "parent of"
- Returns single parent (JJ doesn't support multiple parents in working copy)

#### PR Description Formatting

```python
def format_pr_description(change, all_changes, username, change_to_pr):
    """Format PR description with stack info and cross-references."""
    description = change.description

    # Add stack information
    position = find_position(change, all_changes)
    parent_pr = find_parent_pr(change, change_to_pr)
    child_prs = find_child_prs(change, all_changes, change_to_pr)

    # Format markdown
    return f"""
{description}

---

**Stack Information:**
- **Stack Position:** {position} of {len(all_changes)}
- **Parent PR:** #{parent_pr} (title)
- **Child PRs:** #{child1}, #{child2}

**JJ Change ID:** `{change.change_id}`

---
_This PR is part of a stack managed by jj-stack._
"""
```

### GitHub API Integration

Uses `gh` CLI instead of direct API calls:

**Rationale:**
- Handles authentication automatically
- Simpler than managing API tokens
- Better error messages
- Respects user's GitHub configuration

**Common Operations:**
```bash
# Get username
gh api user -q .login

# List PRs
gh pr list --json number,headRefName --state open

# Create PR
gh pr create -B base -H head -t title -b body

# Update PR
gh pr edit <number> --title <title> --base <base> --body <body>

# Get PR info
gh pr view <number> --json number,title,state,url
```

### JJ Integration

Uses `jj` CLI:

**Common Operations:**
```bash
# Query stack
jj log -r 'trunk()..@' --reversed --no-graph -T '<template>'

# Get parent
jj log -r '<change_id>-' --no-graph -T 'change_id.short()'

# Create bookmark
jj bookmark create <name> -r <change_id>

# Track bookmark with remote
jj bookmark track <name> --remote=origin

# Push bookmark
jj git push --bookmark <name>

# Delete bookmark
jj bookmark delete <name>

# Push deletions
jj git push --deleted
```

## Error Handling Strategy

### Critical Errors (Exit Immediately)

- JJ not available
- GitHub CLI not available or not authenticated
- Not in a JJ repository
- Cannot determine trunk
- Rebase conflicts (sync command)

**Implementation:**
```python
try:
    run(['jj', '--version'])
except:
    print("Error: jj not available")
    sys.exit(1)
```

### Recoverable Errors (Warn and Continue)

- Individual PR creation fails
- Individual PR update fails
- Bookmark push fails

**Implementation:**
```python
try:
    create_pr(...)
except Exception as e:
    print(f"Warning: Failed to create PR: {e}")
    # Continue with next PR
```

### User Errors (Clear Guidance)

- Wrong directory
- Invalid configuration
- Missing trunk definition

**Implementation:**
```python
if result.returncode != 0:
    print("Error: Failed to resolve trunk()")
    print("Hint: Define trunk() in .jj/repo/config.toml")
    sys.exit(1)
```

## Performance Considerations

### Minimize API Calls

**Problem:** GitHub API has rate limits.

**Solution:**
- Batch queries where possible
- Use `gh pr list` once, filter in Python
- Only query individual PR details when needed

### Efficient JJ Queries

**Problem:** Multiple JJ invocations can be slow.

**Solution:**
- Use revsets to query multiple changes at once
- Use templates to get all needed data in one call
- Avoid querying same information multiple times

### Parallel Operations

**Current:** Sequential PR creation/update

**Future Optimization:**
- Could parallelize PR creation using async/threading
- Would need to handle rate limits carefully

## Security Considerations

### Credential Management

- Relies on `gh` CLI for authentication
- No tokens stored in script
- Uses user's existing GitHub authentication

### Command Injection

- All subprocess calls use list form: `['cmd', 'arg1', 'arg2']`
- Never uses shell=True
- Arguments are not interpolated into shell strings

**Safe:**
```python
run(['gh', 'pr', 'create', '-t', user_input])
```

**Unsafe (not used):**
```python
run(f'gh pr create -t "{user_input}"', shell=True)
```

### Bookmark Name Sanitization

```python
def generate_slug(description: str) -> str:
    """Sanitize description to create safe bookmark name."""
    slug = description.lower()
    slug = re.sub(r'[^a-z0-9\s-]', '', slug)  # Remove special chars
    slug = re.sub(r'[-\s]+', '-', slug)        # Normalize hyphens
    slug = slug.strip('-')                     # Trim edges
    return slug
```

## Testing Strategy

### Manual Testing

**Test Repository:** `~/Development/jj-test`

**Test Cases:**
1. Create stack of 3 changes
2. Push stack (creates PRs)
3. Verify bookmarks created with correct format
4. Verify PRs have correct base branches
5. Verify cross-references in PR descriptions
6. Modify middle change
7. Push again (updates, doesn't duplicate)
8. Merge PR
9. Run cleanup (removes bookmark)

### Future: Automated Testing

**Unit Tests:** (not yet implemented)
- Bookmark name generation
- Slug sanitization
- Change ID extraction
- Template parsing

**Integration Tests:** (not yet implemented)
- Full workflow with real GitHub repo
- Mock JJ and gh CLI calls

## Limitations and Future Work

### Current Limitations

1. **No conflict resolution:** If rebase fails, user must resolve manually
2. **No draft PR support:** All PRs created as ready for review
3. **No PR labels/assignees:** Could add automatic labeling
4. **Sequential operations:** Could parallelize for speed
5. **Date in bookmark name:** Changes daily, could use other strategies

### Future Enhancements

**Configuration File:**
```toml
# .jj-stack.toml
[settings]
bookmark_pattern = "{username}/{date}/{slug}-{change_id}"
draft_by_default = false
auto_cleanup = false

[labels]
stack_pr = "stacked-pr"
auto_generated = "auto-generated"
```

**Interactive Mode:**
```bash
jj-stack push --interactive
# Shows preview of PRs to be created
# Asks for confirmation before creating each
```

**Better Branch Visualization:**
```bash
jj-stack tree
# Shows ASCII tree of stack with PR numbers
```

**Smarter Cleanup:**
```bash
jj-stack cleanup --auto
# Automatically cleans up after fetching latest trunk
```

**PR Templates:**
```bash
jj-stack push --template .github/PULL_REQUEST_TEMPLATE.md
# Uses PR template for description
```

## Code Organization

```
jj-stack (single Python file, ~700 lines)
├── Imports and setup
├── Data Models (Change class)
├── Command execution utilities
│   ├── run()
│   └── check_prerequisites()
├── GitHub integration
│   ├── get_github_username()
│   ├── find_pr_for_change()
│   ├── get_pr_info()
│   └── get_pr_state()
├── JJ integration
│   ├── get_stack_changes()
│   ├── get_parent_change_id()
│   ├── get_bookmarks_for_change()
│   └── bookmark_exists()
├── Bookmark management
│   ├── generate_slug()
│   ├── generate_bookmark_name()
│   └── ensure_bookmark()
├── PR operations
│   ├── format_pr_description()
│   ├── create_pr()
│   └── update_pr()
├── Command implementations
│   ├── cmd_push()
│   ├── cmd_status()
│   ├── cmd_sync()
│   ├── cmd_list()
│   └── cmd_cleanup()
└── Main entry point
    └── main()
```

## Dependencies

**Runtime:**
- Python 3 (standard library only)
- `jj` (Jujutsu VCS)
- `gh` (GitHub CLI)

**Python Standard Library:**
- `argparse`: CLI argument parsing
- `json`: JSON parsing for `gh` output
- `re`: Regular expressions for slug generation
- `subprocess`: Running external commands
- `sys`: Exit codes and stderr
- `dataclasses`: Data models
- `datetime`: Timestamp for bookmark names
- `typing`: Type hints
- `collections.defaultdict`: For graph algorithms

**No External Dependencies:** Deliberately kept simple to avoid installation complexity.

## Maintenance

### Code Style

- Type hints for all functions
- Docstrings for all public functions
- Clear variable names
- Comments for non-obvious logic

### Version Control

Located at: `~/bin/jj-stack`

**To version control:**
```bash
git init ~/bin
cd ~/bin
git add jj-stack jj-stack-*.md
git commit -m "Initial version of jj-stack"
```

### Updates

When updating:
1. Test with `~/Development/jj-test` repo
2. Verify all commands work
3. Update documentation if behavior changes
4. Update version in README

## References

- [Jujutsu Documentation](https://github.com/martinvonz/jj)
- [GitHub CLI Documentation](https://cli.github.com/)
- [GitHub REST API](https://docs.github.com/en/rest)

---

**Last Updated:** 2026-02-18
**Version:** 1.0.0

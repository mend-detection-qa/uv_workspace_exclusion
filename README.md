# Phase 4 Test: Workspace Exclusion

## Test ID
T-P4-023

## Category
Workspace - Member Exclusion

## Priority
P1

## Description
This fixture tests UV's workspace `exclude` functionality, which allows excluding specific directories from workspace membership even if they match glob patterns in `members`. This is based on GitHub issue [#7071](https://github.com/astral-sh/uv/issues/7071).

## GitHub Issue Context

### Original Issue (#7071):
**Problem**: When using glob patterns with nested directories, it was unclear how exclusion worked.

**Configuration attempted**:
```toml
[tool.uv.workspace]
members = ["projects/*", "projects/b/*"]
exclude = ["projects/b"]
```

**Expected**: `projects/b` excluded, but `projects/b/ba` and `projects/b/bb` included.

**Resolution** (PR #7175): `exclude` is a **post-filter** that operates on the expanded `members` list. To exclude a directory, it must match at the appropriate depth.

## How Workspace Exclusion Works

### Key Rules:
1. **`members` expands first**: Glob patterns are expanded to find all matching directories
2. **`exclude` filters second**: Exclusion patterns remove matches from the expanded list
3. **Pattern depth matters**: `exclude = ["projects/b"]` excludes that exact path
4. **No regex needed**: Simple glob patterns work

### Post-Filter Behavior:
```toml
members = ["projects/*"]
# Expands to: [projects/a, projects/b, projects/c, projects/excluded1, projects/excluded2]

exclude = ["projects/excluded1", "projects/excluded2"]
# Removes: projects/excluded1, projects/excluded2

# Final members: [projects/a, projects/b, projects/c]
```

## File Structure

```
workspace_exclusion/
├── pyproject.toml                # Root with exclusions
├── README.md                     # This file
├── projects/
│   ├── a/                       # ✓ Included (matches projects/*)
│   │   └── pyproject.toml
│   ├── b/                       # No pyproject.toml (container only)
│   │   ├── README.md
│   │   ├── ba/                  # ✓ Included (matches projects/b/*)
│   │   │   └── pyproject.toml
│   │   └── bb/                  # ✓ Included (matches projects/b/*)
│   │       └── pyproject.toml
│   ├── c/                       # ✓ Included (matches projects/*)
│   │   └── pyproject.toml
│   ├── excluded1/               # ✗ Excluded (explicitly)
│   │   └── pyproject.toml
│   └── excluded2/               # ✗ Excluded (explicitly)
│       └── pyproject.toml
└── docs/                        # ✗ Excluded (in exclude list)
```

## Workspace Configuration

### Root pyproject.toml:
```toml
[tool.uv.workspace]
members = [
    "projects/*",      # Include all direct children
    "projects/b/*",    # Include nested projects in b/
]
exclude = [
    "projects/excluded1",
    "projects/excluded2",
    "docs",
]
```

### Expansion Process:
```
Step 1 - Expand members:
  projects/* → [projects/a, projects/b, projects/c, projects/excluded1, projects/excluded2]
  projects/b/* → [projects/b/ba, projects/b/bb]
  Combined: [projects/a, projects/b, projects/c, projects/excluded1, projects/excluded2,
             projects/b/ba, projects/b/bb]

Step 2 - Apply exclude filter:
  Remove: projects/excluded1
  Remove: projects/excluded2
  (projects/b has no pyproject.toml, ignored automatically)

Step 3 - Final workspace members:
  ✓ projects/a
  ✓ projects/b/ba
  ✓ projects/b/bb
  ✓ projects/c
```

## Expected Behavior

### With Correct Exclusion (Success):
```bash
$ uv lock

Workspace members detected:
  ✓ project-a (projects/a)
  ✓ project-ba (projects/b/ba)
  ✓ project-bb (projects/b/bb)
  ✓ project-c (projects/c)

Excluded:
  ✗ projects/excluded1 (in exclude list)
  ✗ projects/excluded2 (in exclude list)
  ℹ projects/b (no pyproject.toml)
  ℹ docs (in exclude list, not a project anyway)

Resolved 45 packages in 1.2s

$ uv tree
workspace-exclusion
├── project-a (workspace)
├── project-ba (workspace)
│   └── project-a (workspace)
├── project-bb (workspace)
│   ├── project-a (workspace)
│   └── project-ba (workspace)
└── project-c (workspace)
    └── project-ba (workspace)
```

### Without Exclusion (Would Include All):
```toml
[tool.uv.workspace]
members = ["projects/*", "projects/b/*"]
# No exclude

# Would try to include:
# projects/excluded1 ✓
# projects/excluded2 ✓
```

### Wrong Pattern Depth:
```toml
[tool.uv.workspace]
members = ["projects/*"]
exclude = ["excluded1"]  # Wrong! Doesn't match "projects/excluded1"

# excluded1 wouldn't be filtered out
# Need: exclude = ["projects/excluded1"]
```

## Test Scenarios

### Scenario 1: Verify Exclusion
```bash
$ cd workspace_exclusion
$ uv lock

# Should NOT see excluded1 or excluded2 in workspace
# Should see: a, ba, bb, c
```

### Scenario 2: Check Workspace List
```bash
$ uv run python -c "import sys; print([p for p in sys.path if 'projects' in p])"

# Should show:
# .../projects/a
# .../projects/b/ba
# .../projects/b/bb
# .../projects/c

# Should NOT show:
# .../projects/excluded1
# .../projects/excluded2
```

### Scenario 3: Verify Dependencies
```bash
$ uv tree

# project-bb depends on project-a and project-ba (both workspace members)
# project-c depends on project-ba (workspace member)
# All should resolve correctly
```

### Scenario 4: Try Installing Excluded Package
```bash
$ cd projects/excluded1
$ uv sync

# This package can still be used standalone
# Just not part of the root workspace
```

## Success Criteria

- [ ] Workspace detects 4 members: a, ba, bb, c
- [ ] excluded1 and excluded2 NOT included in workspace
- [ ] Nested packages (ba, bb) included correctly
- [ ] Cross-workspace dependencies resolve
- [ ] Lock file generation succeeds
- [ ] No errors about missing pyproject.toml in excluded paths
- [ ] projects/b (no pyproject.toml) handled gracefully

## Use Cases

### Use Case 1: Exclude Experimental Projects
```toml
members = ["projects/*"]
exclude = [
    "projects/experimental",
    "projects/deprecated",
]
```

### Use Case 2: Exclude Template Directories
```toml
members = ["packages/*"]
exclude = [
    "packages/template",
    "packages/_scaffold",
]
```

### Use Case 3: Exclude Broken Projects
```toml
members = ["services/*"]
exclude = [
    "services/old-service",  # Not compatible
    "services/wip-service",  # Work in progress
]
```

### Use Case 4: Nested Workspaces
```toml
members = [
    "apps/*",
    "libs/*",
    "internal/tools/*",  # Nested under internal/
]
exclude = [
    "internal/docs",  # Exclude docs from internal/
]
```

## Common Issues

### Issue 1: Wrong exclude pattern depth
```toml
# Wrong:
members = ["projects/*"]
exclude = ["excluded1"]  # Doesn't match!

# Correct:
members = ["projects/*"]
exclude = ["projects/excluded1"]
```

### Issue 2: Excluding parent directory
```toml
members = ["projects/*", "projects/b/*"]
exclude = ["projects/b"]  # Doesn't exclude ba and bb!

# projects/b has no pyproject.toml anyway
# ba and bb are still included via "projects/b/*"
```

### Issue 3: Glob in exclude
```toml
# Exclude supports globs too:
exclude = ["projects/temp-*"]  # Excludes temp-1, temp-2, etc.
```

## Edge Cases

### Edge Case 1: Directory without pyproject.toml
```
projects/b/  # No pyproject.toml
├── ba/      # Has pyproject.toml
└── bb/      # Has pyproject.toml

# projects/b not a workspace member (no pyproject.toml)
# ba and bb ARE workspace members (have pyproject.toml)
```

### Edge Case 2: Overlapping patterns
```toml
members = ["projects/*", "projects/special/*"]
exclude = ["projects/special"]

# special/ would be excluded
# But special/sub would be included (no overlap)
```

### Edge Case 3: Excluding non-existent path
```toml
exclude = ["projects/nonexistent"]

# No error - just ignored
```

## Mend Integration Notes

**Scanning Workspaces with Exclusions:**

Mend should:
- Respect UV's exclude configuration
- Scan only included workspace members
- Report dependencies from included members only
- Not report errors for excluded members
- Handle nested workspace members correctly

**Expected Mend Behavior:**
```
Workspace members scanned:
✓ project-a
✓ project-ba
✓ project-bb
✓ project-c

Excluded (not scanned):
✗ project-excluded1
✗ project-excluded2

Total: 4 workspace members
```

## Related Tests
- T-P2-008: Workspace Monorepo
- T-P4-013: Workspace with Multiple UV Version Features
- T-P4-021: Workspace UV Version Requirements

## Documentation Links
- GitHub Issue #7071: https://github.com/astral-sh/uv/issues/7071
- GitHub PR #7175: https://github.com/astral-sh/uv/pull/7175
- UV Workspaces: https://docs.astral.sh/uv/concepts/workspaces/
- UV Configuration: https://docs.astral.sh/uv/configuration/workspaces/

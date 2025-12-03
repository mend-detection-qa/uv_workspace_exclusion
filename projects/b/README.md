# Projects B Container

This directory does NOT have a pyproject.toml.
It's just a container for nested workspace members:
- ba/
- bb/

The workspace configuration includes:
- members = ["projects/b/*"]  # Include ba and bb
- No pyproject.toml in this directory itself

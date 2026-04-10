# Global CLAUDE.md

## Installing dependencies

- `~/.vps-dotfiles/install` is an idempotent script used to install dependencies and tools you may need for development.
- If you discover that you need a new dependency or tool, instead of installing it on the fly, edit `~/.vps-dotfiles/install` to idempotently install the new requirement, and run the script.

## Software development guidelines

- The `main` branch should always remain free of changes. Never commit nor push to the `main` branch.
- All work must be done in a new worktree, on its own branch.

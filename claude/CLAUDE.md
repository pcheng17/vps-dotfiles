# Global CLAUDE.md

## Installing dependencies

- `~/.vps-dotfiles/install` is an idempotent script used to install dependencies and tools you may need for development.
- If you discover that you need a new dependency or tool, instead of installing it on the fly, edit `~/.vps-dotfiles/install` to idempotently install the new requirement, and run the script.

## Workflow

- Always create a todo list of tasks so I can keep track of what you're doing.

## Software development guidelines

- Always create a new branch and worktree for new changes.
- The `main` branch should always remain free of changes.
- Never commit nor push to the `main` branch.
- If the user asks to tackle multiple tasks at once that can be done in parallel, use separate agents for each task, each working on its own worktree.

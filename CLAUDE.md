# vps-dotfiles

This repo lives at `~/.vps-dotfiles`.

## Structure

- `install` -- The main entry point. An idempotent bash script that bootstraps a VPS with all required tools and configuration.
- `claude/CLAUDE.md` -- The user's global Claude Code instructions (symlinked to `~/.claude/CLAUDE.md`).
- `claude/settings.json` -- Claude Code settings (symlinked to `~/.claude/settings.json`).

## Adding dependencies

- For Homebrew packages, add them to the `PACKAGES` array at the top of `install`.
- For tools that require custom installation, add a new idempotent section to `install` following the existing pattern (check if already installed, skip if so, install otherwise).

## Important

The install script must remain idempotent -- every section must be safe to re-run.

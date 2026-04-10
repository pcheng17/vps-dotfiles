# vps-dotfiles

Bootstrap a fresh Ubuntu VPS with development tools and configuration.

## Usage

```bash
git clone <repo-url> ~/.vps-dotfiles
cd ~/.vps-dotfiles
./install
```

## What it sets up

- **Homebrew** as the package manager
- **Dev packages**: cmake, gcc, go, ninja, neovim, fzf, fd, gh, ripgrep, rust, lazygit, lld, llvm, uv, and more
- **nvm** + **Node.js 22**
- **Claude Code** CLI
- **Docker Engine**
- **Claude Code config**: symlinks `claude/CLAUDE.md` and `claude/settings.json` into `~/.claude/`

## Idempotent

The install script is idempotent -- safe to re-run at any time. Each section
checks whether its target is already installed before doing anything.

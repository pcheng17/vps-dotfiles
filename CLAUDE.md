# vps-dotfiles

This repo lives at `~/.vps-dotfiles`.

## Fresh VPS bootstrap

SSH in as root, then run:

```
bash <(curl -fsSL https://raw.githubusercontent.com/pcheng17/vps-dotfiles/main/bootstrap)
```

Bootstrap (running as root) will:
1. Prompt for a username to create (or reuse if it exists), add them to `sudo`, and set a password.
2. Copy root's `~/.ssh/authorized_keys` to the new user so you can SSH in as them.
3. Clone this repo into that user's `~/.vps-dotfiles`.
4. Run `install --root` (apt, gh, Tailscale, Docker).
5. Re-exec `install --user` as the new user (brew + formulae, uv, nvm, Claude Code, config symlinks).

Afterwards, log out and SSH back in as the new user, then run `gh auth login` once.

## Structure

- `bootstrap` -- One-command entry point for a fresh VPS. Must run as root. Creates a target user, then runs both `install --root` and `install --user` (the latter as the new user).
- `install` -- Idempotent bash script that provisions the VPS. Requires a mode flag:
    - `./install --root` (run as root) -- apt packages, GitHub CLI, Tailscale, Docker
    - `./install --user` (run as your user) -- Homebrew + formulae, uv/Python, nvm/Node, Claude Code, shell/git/Claude config symlinks
    - `sudo ./install --all` -- runs both phases; `--user` is re-invoked as `$SUDO_USER`
- `setup-swap` -- Optional standalone script. Provisions a swapfile (default 8G at `/swapfile`), enables it, adds it to `/etc/fstab`, and sets `vm.swappiness=10`. Run as root. Override with `SWAP_SIZE` / `SWAP_FILE` / `SWAPPINESS` env vars. Idempotent. Not invoked by `bootstrap` or `install` -- run manually on hosts that need swap.
- `claude/CLAUDE.md` -- The user's global Claude Code instructions (symlinked to `~/.claude/CLAUDE.md`).
- `claude/settings.json` -- Claude Code settings (symlinked to `~/.claude/settings.json`).

## Adding dependencies

- Homebrew (user) packages: add to the `BREW_PACKAGES` array at the top of `install`.
- apt (root) packages: add to the `APT_PACKAGES` array at the top of `install`.
- Tools that require custom installation: add a new idempotent block to the relevant `install_root` or `install_user` function (check if already installed, skip if so, install otherwise).

## Important

The install script must remain idempotent -- every section must be safe to re-run.

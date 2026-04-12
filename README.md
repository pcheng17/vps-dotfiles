# vps-dotfiles

Bootstrap a fresh Ubuntu VPS with development tools and configuration.

## Usage

SSH in as `root` on a fresh VPS and run:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/pcheng17/vps-dotfiles/main/bootstrap)
```

Bootstrap will:

1. Prompt for a username to create (or reuse if it already exists), add them to `sudo`, and set a password.
2. Prompt for `git user.name` and `git user.email` (passed through to the user phase as `GIT_NAME` / `GIT_EMAIL`).
3. Copy root's `~/.ssh/authorized_keys` to the new user so you can SSH in as them.
4. Clone this repo into `~<user>/.vps-dotfiles`.
5. Run `install --root` -- apt packages, GitHub CLI, Tailscale, Docker.
6. Re-exec `install --user` as the new user -- Homebrew + formulae, uv/Python, nvm/Node, Claude Code, config symlinks.

Afterwards, log out, SSH back in as the new user, and run `gh auth login` once.

## Manual invocation

If you've already cloned the repo and just want to run a phase:

```bash
sudo ./install --root                                        # system packages (run as root)
GIT_NAME="Your Name" GIT_EMAIL="you@example.com" \
    ./install --user                                         # user-level tools (run as your user)
sudo ./install --all                                         # both phases (requires SUDO_USER)
```

`GIT_NAME` / `GIT_EMAIL` are optional on `--user` -- if unset, `user.name` and
`user.email` are left untouched (useful for re-runs after bootstrap has already
set them).

## What it sets up

**Root phase (`--root`)**

- apt packages: clang, cmake, build-essential, ninja-build
- GitHub CLI (`gh`) via the official apt repo
- Tailscale (via `tailscale.com/install.sh`)
- Docker Engine (via `get.docker.com`); target user added to the `docker` group

**User phase (`--user`)**

- Homebrew + formulae: lazygit, neovim, fzf, fd, opencode, ripgrep, uv
- Python 3.14 via `uv`
- `nvm` + Node.js 22
- Claude Code CLI
- Shell config symlinks: `~/.bashrc`, `~/.bash_aliases`
- Git config (user, editor, credential helper using `gh`)
- Claude Code config symlinks: `~/.claude/CLAUDE.md`, `~/.claude/settings.json`

## Optional: swapfile

Small VPSes often lack swap and get OOM-killed under transient memory spikes.
`setup-swap` provisions a swapfile, enables it, persists it via `/etc/fstab`,
and lowers `vm.swappiness` to 10. Idempotent.

```bash
sudo ./setup-swap                       # 8G swapfile at /swapfile
sudo SWAP_SIZE=16G ./setup-swap         # custom size
sudo SWAP_FILE=/mnt/swap ./setup-swap   # custom path
```

## Idempotent

Every section of both `bootstrap` and `install` is safe to re-run. Re-running
`bootstrap` fast-forward-pulls the repo and skips anything already installed.

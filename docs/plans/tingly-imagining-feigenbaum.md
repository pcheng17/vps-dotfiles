# Split `install` into root and user modes

## Context

Today `install` mixes system-level work (apt packages, Docker, Tailscale — all
requiring `sudo`) with user-level work (Homebrew, nvm/Node, uv/Python, Claude
Code, shell + git + Claude config symlinks under `$HOME`). Running the whole
thing as a single user is awkward: either you run as root and Homebrew/nvm end
up in root's `$HOME` (wrong), or you run as your user and it keeps prompting
for `sudo` mid-script.

The user wants to pick a mode explicitly: `--root` for apt/Docker/Tailscale,
`--user` for everything that lives under `$HOME`. Must stay idempotent.

There's also a bootstrap-ordering concern: on a fresh VPS we need to clone
this repo before running the script, and the current git config references
`$(brew --prefix)/bin/gh` — meaning `gh` was expected from Homebrew, which
comes from `--user` and thus runs *after* the clone. Resolution: move `gh`
out of `BREW_PACKAGES` and install it via apt in `--root` (using the official
GitHub CLI apt repo), and fix the git config to use plain `gh` on `PATH`.
The initial clone itself uses plain `git clone https://...` — `git` is
preinstalled on Ubuntu, and the repo is public, so no auth needed at clone
time.

## Approach

Add a single required flag that selects the execution path. No other
behavioral changes — we're partitioning existing sections, not rewriting them.

### CLI

```
./install --root         # apt, Tailscale, Docker (must be run as root / via sudo)
./install --user         # shell/claude/git config, brew + pkgs, uv, nvm, claude code
./install --all          # runs --root then --user (errors unless invoked via sudo with SUDO_USER set)
./install                # prints usage and exits 1
```

Parsed with a simple `case "$1" in` block near the top. Unknown flag → usage + exit 1.

### Section assignment

**Root mode** (requires `EUID == 0`; error out otherwise):
- apt packages block (drop the `sudo` prefix on `apt-get` since we're already root). Remove `gh` considerations — handled in its own block below.
- **GitHub CLI** (new block, runs before the general apt block so its repo is available — actually fine either way since it's a separate `apt update`). Idempotent guard: `command -v gh &>/dev/null`. Commands (no `sudo`, since we're root):
  ```
  apt-key adv --keyserver keyserver.ubuntu.com --recv-key C99B11DEB97541F0
  apt-add-repository -y https://cli.github.com/packages
  apt update
  apt install -y gh
  ```
  Note: `apt-key` is deprecated on newer Ubuntu but the user requested these exact commands; keep as-is. `-y` added to `apt-add-repository` and `apt install` so the script doesn't prompt.
- Tailscale install (official script works fine under root)
- Docker install — the `usermod -aG docker` line becomes `usermod -aG docker "${SUDO_USER:-$USER}"` so that when invoked via `sudo ./install --root`, the *invoking* user gets added to the docker group, not root. If `SUDO_USER` is unset and we're root, log a warning and skip the `usermod`.

**User mode** (refuse if `EUID == 0` to prevent polluting `/root`):
- Claude Code config symlinks (`$HOME/.claude/*`)
- Shell config symlinks (`$HOME/.bashrc`, `$HOME/.bash_aliases`)
- Homebrew install + `BREW_PACKAGES` install loop. **Remove `gh` from `BREW_PACKAGES`** — it's now apt-installed in `--root`.
- Git config block. **Change** line 32 from `"!$(brew --prefix)/bin/gh auth git-credential"` to `"!gh auth git-credential"` (plain `gh` on `PATH`, since gh now comes from `/usr/bin/gh` via apt). This also removes the implicit dependency on brew being installed first, so the git config block can move anywhere in `install_user`.
- `uv python install 3.14`
- nvm + Node.js 22
- Claude Code (`curl ... claude.ai/install.sh | bash`)

**Shared** (kept at top of script, above the mode dispatch):
- `set -e`
- `BREW_PACKAGES` / `APT_PACKAGES` arrays
- `SCRIPT_DIR`
- `log_info` / `log_success` helpers
- New: `log_error`, `usage()`

### Structure sketch

```bash
#!/bin/bash
set -e

BREW_PACKAGES=(...)
APT_PACKAGES=(...)
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

log_info()    { echo "[INFO] $*"; }
log_success() { echo "[OK]   $*"; }
log_error()   { echo "[ERR]  $*" >&2; }

usage() {
    cat <<EOF
Usage: $0 <--root|--user|--all>
  --root   Install system packages (apt, Docker, Tailscale). Run as root.
  --user   Install user-level tools (brew, nvm, uv, configs). Run as your user.
  --all    Run --root then --user. Invoke via sudo; SUDO_USER must be set.
EOF
}

install_root() { ... }   # apt + tailscale + docker
install_user() { ... }   # configs + brew + uv + nvm + claude code

case "${1:-}" in
    --root) [[ $EUID -eq 0 ]] || { log_error "--root must run as root"; exit 1; }
            install_root ;;
    --user) [[ $EUID -ne 0 ]] || { log_error "--user must not run as root"; exit 1; }
            install_user ;;
    --all)  [[ $EUID -eq 0 && -n "${SUDO_USER:-}" ]] || { log_error "--all requires sudo with SUDO_USER"; exit 1; }
            install_root
            sudo -u "$SUDO_USER" -H "$0" --user ;;
    *)      usage; exit 1 ;;
esac

log_success "Done!"
```

### Files modified / created

- `/Users/pcheng/dev/vps-dotfiles/install` — refactor into `--root`/`--user`/`--all` modes; move `gh` to root-apt block; fix git config credential helper
- `/Users/pcheng/dev/vps-dotfiles/bootstrap` — **new**, executable, the one-command entry point
- `/Users/pcheng/dev/vps-dotfiles/CLAUDE.md` — update "Structure" section: document `bootstrap` and `install --root|--user|--all`

### Fresh-VPS bootstrap: one command

Add a new `bootstrap` script at the repo root. The fresh-VPS experience
becomes a single command:

```
bash <(curl -fsSL https://raw.githubusercontent.com/pcheng17/vps-dotfiles/main/bootstrap)
```

Why `bash <(curl ...)` and not `curl ... | bash`? The pipe form takes over
bash's stdin, which breaks any `sudo` password prompt inside the script.
Process substitution keeps stdin attached to the terminal, so `sudo` inside
`bootstrap` can prompt normally.

`bootstrap` contents:

```bash
#!/bin/bash
set -e

REPO_URL="https://github.com/pcheng17/vps-dotfiles"
TARGET="$HOME/.vps-dotfiles"

log_info()    { echo "[INFO] $*"; }
log_success() { echo "[OK]   $*"; }
log_error()   { echo "[ERR]  $*" >&2; }

if [[ $EUID -eq 0 ]]; then
    log_error "Run bootstrap as your normal user, not root. It will use sudo where needed."
    exit 1
fi

# git is preinstalled on every Ubuntu VPS I've seen, but be defensive.
if ! command -v git &>/dev/null; then
    log_info "Installing git..."
    sudo apt-get update -qq
    sudo apt-get install -y git
fi

# Clone (or update if re-running)
if [[ -d "$TARGET/.git" ]]; then
    log_info "Updating existing clone at $TARGET..."
    git -C "$TARGET" pull --ff-only
else
    log_info "Cloning $REPO_URL into $TARGET..."
    git clone "$REPO_URL" "$TARGET"
fi

cd "$TARGET"
sudo ./install --root
./install --user

log_success "Bootstrap complete. Run 'gh auth login' to authenticate GitHub CLI."
```

Idempotent: re-running just does a fast-forward pull and lets `install --root`
/ `install --user` skip already-installed items.

Manual sequence (equivalent, for when you've already cloned):

```
cd ~/.vps-dotfiles
sudo ./install --root
./install --user
gh auth login    # one-time, after --root installs gh
```

## Verification

1. **Usage / arg parsing** (no side effects):
   - `./install` → prints usage, exits 1
   - `./install --bogus` → prints usage, exits 1
2. **Guard rails**:
   - `./install --root` as non-root → errors cleanly
   - `./install --user` as root → errors cleanly
3. **Idempotence** (the repo's invariant from `CLAUDE.md`):
   - `sudo ./install --root` twice → second run reports everything already installed, no changes
   - `./install --user` twice → same
4. **End-to-end on a fresh VPS**:
   - `sudo ./install --root` → `apt list --installed | grep clang`, `command -v docker`, `command -v tailscale`, `command -v gh` (and `gh --version` runs), `id $SUDO_USER | grep docker`
   - `./install --user` → `command -v brew`, `brew list | grep lazygit` (and `gh` should NOT be in brew list), `nvm ls 22`, `uv python find 3.14`, `command -v claude`, `ls -l ~/.claude/CLAUDE.md ~/.bashrc` (both symlinks), `git config --global credential.https://github.com.helper` returns `!gh auth git-credential`
5. **Docker group assignment** under `--all`: after `sudo ./install --all`, `id "$SUDO_USER"` lists `docker`.

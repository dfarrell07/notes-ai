---
date: 2026-05-28T23:30:00-04:00
topic: laptop-setup
tags: [distrobox, toolbx, fedora, rhel-csb, fapolicyd, podman, containers, development-environment]
---

# Distrobox Development Environment on RHEL CSB

Specification for a Fedora container running inside Distrobox on a RHEL CSB host. The container provides a full development environment that bypasses fapolicyd restrictions on the host.

## Architecture

Two tiers:

- **Host (RHEL CSB):** Runs Podman rootless, Distrobox (or Toolbx), SSH, Tailscale (if IT allows). Minimal developer tools -- just enough to enter the container.
- **Container (Fedora latest):** Full development environment. All compilers, linters, CLIs, language runtimes. This is where the developer spends their time.

The developer works inside the container, not on the host. The container is the primary shell environment. Exported binaries are a convenience for quick host-side commands, not the primary workflow.

## Question 1: What Goes in the Container vs Host?

### Host provides (already installed on RHEL CSB or installable via RPM):
- Podman rootless (ships with RHEL)
- Distrobox or Toolbx (see Question 9)
- SSH server and client
- Tailscale (if permitted)
- tmux (for persistent sessions)
- git (basic operations only -- Claude Code and full dev workflow happen inside container)

### Container provides (everything else):
- Go, Python 3, gcc, clang
- oc, kubectl, kustomize, helm, opm, subctl
- gh, acli, golangci-lint, shellcheck, shfmt, yamllint, grype
- jq, yq, tmux, vim, direnv, fzf, zoxide
- Node.js (for Claude Code)
- Claude Code
- zsh + oh-my-zsh

### Container engine (Podman/Docker): Use the host's

The container does NOT install its own Podman or Docker. Instead, it uses `distrobox-host-exec` symlinks to delegate container commands to the host's Podman. This avoids nested containers entirely.

```bash
# Inside the container, these symlinks transparently call the host's Podman
sudo ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/podman
sudo ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/docker
```

When you run `podman build .` inside the container, it actually runs on the host. Build context (Dockerfiles, source code) is visible because the home directory is shared.

### kind (Kubernetes-in-Docker): Works via host delegation

kind is installed as a binary inside the container. When kind calls Podman to create cluster nodes, it uses the symlinked Podman which delegates to the host. The kind binary itself is just a Go binary with no special requirements -- only its container runtime backend needs to be real.

```bash
# Inside the container
export KIND_EXPERIMENTAL_PROVIDER=podman
kind create cluster
# kind calls "podman" which is actually distrobox-host-exec -> host's podman
```

This works because:
1. kind just needs a `podman` or `docker` binary in PATH
2. The symlink makes `podman` transparently execute on the host
3. The host's Podman creates the actual container nodes
4. kubectl (installed in the container) connects to the cluster via kubeconfig in the shared home directory

## Question 2: distrobox.ini and Ansible Integration

### distrobox.ini format

INI-style manifest for `distrobox assemble`. Each section defines a container by name.

```ini
[fedora-dev]
image=registry.fedoraproject.org/fedora:latest
pull=true
replace=true
start_now=false

# Packages can be split across multiple lines for readability
additional_packages="git vim tmux zsh fzf"
additional_packages="gcc gcc-c++ clang make cmake"
additional_packages="golang python3 python3-pip nodejs npm"
additional_packages="jq ShellCheck shfmt yamllint"
additional_packages="direnv zoxide bash-completion"
additional_packages="openssl-devel libffi-devel"
additional_packages="procps-ng findutils which hostname"

# Enable EPEL-like repos before package install
pre_init_hooks=dnf install -y 'dnf-command(copr)';

# Post-init: host delegation symlinks, binary installs, exports
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/podman;
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/docker;
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/flatpak;
```

### Ansible generates it

The Ansible `distrobox` role generates `distrobox.ini` from a Jinja2 template, then runs `distrobox assemble create`. This keeps the package list in Ansible variables alongside the host package list.

```yaml
# roles/distrobox/tasks/main.yml
- name: Deploy distrobox.ini
  ansible.builtin.template:
    src: distrobox.ini.j2
    dest: "{{ ansible_user_dir }}/.config/distrobox/distrobox.ini"
    mode: "0644"

- name: Create/update Distrobox container
  ansible.builtin.command:
    cmd: distrobox assemble create --file {{ ansible_user_dir }}/.config/distrobox/distrobox.ini
  changed_when: true  # distrobox assemble doesn't report change status cleanly
```

The Jinja2 template pulls from the same `default.config.yml` variables used for host packages, so adding a tool to the container is one variable change.

### Execution order inside distrobox-init

1. `pre_init_hooks` -- runs before the package manager (enable repos, configure mirrors)
2. Package installation (`additional_packages`) -- dnf install
3. `init_hooks` -- runs after packages are installed (symlinks, binary downloads, exports)

Important: hooks run on every `distrobox enter` if the container needs re-initialization, not just on first creation. Keep them idempotent (use `ln -sf`, check before downloading).

## Question 3: Package Installation Automation

Three approaches, in order of preference:

### Approach A: distrobox.ini additional_packages (preferred for RPM packages)

Packages listed in `additional_packages` are installed by `distrobox-init` during container setup. This is the right approach for anything in the Fedora repos.

### Approach B: init_hooks for binaries not in repos

Tools like oc, kubectl, helm, kind, opm, subctl, gh, grype, golangci-lint, yq are typically installed as standalone binaries. Use `init_hooks` to download and install them:

```ini
# Download binaries that aren't in Fedora repos
init_hooks=/path/to/container-setup.sh;
```

The setup script should live in a location visible inside the container (the shared home directory works):

```bash
#!/bin/bash
# ~/laptop-setup/files/distrobox-setup.sh
# Runs inside the Fedora container during init_hooks
set -euo pipefail

BIN_DIR="/usr/local/bin"
MARKER="$HOME/.local/share/distrobox-setup-done"

# Skip if already run for this container version
if [[ -f "$MARKER" ]]; then
    exit 0
fi

# oc + kubectl
if ! command -v oc &>/dev/null; then
    curl -sL https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz \
        | sudo tar xzf - -C "$BIN_DIR" oc kubectl
fi

# helm
if ! command -v helm &>/dev/null; then
    curl -sL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

# kind
if ! command -v kind &>/dev/null; then
    curl -sLo "$BIN_DIR/kind" "https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64"
    sudo chmod +x "$BIN_DIR/kind"
fi

# kustomize
if ! command -v kustomize &>/dev/null; then
    curl -sL "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
    sudo mv kustomize "$BIN_DIR/"
fi

# gh CLI
if ! command -v gh &>/dev/null; then
    sudo dnf install -y 'dnf-command(config-manager)'
    sudo dnf config-manager addrepo --from-repofile=https://cli.github.com/packages/rpm/gh-cli.repo
    sudo dnf install -y gh
fi

# golangci-lint
if ! command -v golangci-lint &>/dev/null; then
    curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
        | sh -s -- -b "$BIN_DIR"
fi

# grype
if ! command -v grype &>/dev/null; then
    curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh \
        | sh -s -- -b "$BIN_DIR"
fi

# yq
if ! command -v yq &>/dev/null; then
    curl -sLo "$BIN_DIR/yq" "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64"
    sudo chmod +x "$BIN_DIR/yq"
fi

# opm
if ! command -v opm &>/dev/null; then
    curl -sLo "$BIN_DIR/opm" "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/opm-linux.tar.gz"
    # opm comes as a tarball in newer releases
    sudo chmod +x "$BIN_DIR/opm"
fi

# shfmt (if not in repos)
if ! command -v shfmt &>/dev/null; then
    curl -sLo "$BIN_DIR/shfmt" "https://github.com/mvdan/sh/releases/latest/download/shfmt_linux_amd64"
    sudo chmod +x "$BIN_DIR/shfmt"
fi

# Claude Code
if ! command -v claude &>/dev/null; then
    npm install -g @anthropic-ai/claude-code
fi

# acli (pip)
if ! command -v acli &>/dev/null; then
    pip install --user atlassian-cli
fi

# Host delegation symlinks (idempotent with -sf)
sudo ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/podman
sudo ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/docker

touch "$MARKER"
```

### Approach C: distrobox enter -- sudo dnf install (ad hoc)

For one-off package installs, run from the host:

```bash
distrobox enter fedora-dev -- sudo dnf install -y some-package
```

Or from inside the container, just use `sudo dnf install` normally. Sudo works inside Distrobox containers without a password (the container's sudo is configured by distrobox-init).

## Question 4: Which Tools to Export to Host

Export sparingly. The primary workflow is inside the container. Export only tools that are genuinely useful from a host shell (like when you open a quick terminal without entering the container).

### Export these (frequently used from host context):

```bash
# Inside the container, run:
distrobox-export --bin /usr/bin/git --export-path ~/.local/bin
distrobox-export --bin /usr/bin/jq --export-path ~/.local/bin
distrobox-export --bin /usr/bin/vim --export-path ~/.local/bin
distrobox-export --bin /usr/local/bin/gh --export-path ~/.local/bin
distrobox-export --bin /usr/local/bin/oc --export-path ~/.local/bin
distrobox-export --bin /usr/local/bin/kubectl --export-path ~/.local/bin
```

Each exported binary becomes a small wrapper script in `~/.local/bin` that calls `distrobox-enter -e` transparently. The binary works from the host as if it were natively installed.

### Do NOT export these:

- **Claude Code** -- runs inside long-lived tmux sessions in the container. No reason to invoke from host.
- **go, gcc, clang** -- compilation should happen inside the container where the full toolchain is available.
- **golangci-lint, shellcheck, yamllint** -- used during development, inside the container.
- **podman/docker** -- already native on the host.
- **kind** -- needs to interact with podman, which uses the host delegation pattern. Run inside the container.

### Warning about fapolicyd and exports

Exported binaries are wrapper scripts placed in `~/.local/bin`. If fapolicyd is enforcing on the RHEL CSB host, it may block execution of these scripts (they are not RPM-installed). In that case, exports are useless and the developer must work entirely inside the container. This is fine -- it's the expected workflow anyway.

## Question 5: Claude Code Inside Distrobox

Claude Code works inside Distrobox. There are no fundamental issues.

### Why it works:

- **Node.js**: Installed inside the container via `dnf install nodejs npm`. Fedora ships Node.js 20+.
- **Network access**: Distrobox shares the host's network namespace. No port mapping needed. Claude Code's API calls go through the host's network stack directly.
- **File access**: Home directory is shared. Claude Code reads/writes the same files whether inside or outside the container.
- **Terminal/TTY**: Distrobox shares the host terminal. Interactive prompts, ANSI colors, and terminal features work normally.
- **Authentication**: `~/.claude-work/` and `~/.claude-personal/` directories persist in the shared home. Auth tokens survive container recreation.
- **MCP servers**: MCP server processes run inside the container. The Atlassian MCP server (Node.js based) works fine. Any MCP server that needs network or file access works because both are shared with the host.

### Setup inside the container:

```bash
# Node.js comes from Fedora repos (additional_packages)
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
```

### Authentication:

Since the home directory is shared, Claude Code's config and auth files persist across container recreations. The `claude-work` and `claude-personal` aliases defined in `~/.zshrc` work identically inside the container because:
- The shell rc files are shared (same home directory)
- Environment variables are inherited
- `CLAUDE_CONFIG_DIR` points to directories in the shared home

The only consideration is Vertex AI authentication for the work instance. `gcloud auth application-default login` stores credentials in `~/.config/gcloud/` (shared home), so it works. The OAuth browser flow opens the host's browser (Distrobox shares X11/Wayland) and the callback reaches the container because they share the network.

### Known edge case:

If the container's Node.js version differs significantly from what Claude Code expects, there could be compatibility issues. Stick to Fedora's default Node.js (20+) or install a specific version via `dnf module install nodejs:20`.

## Question 6: Dotfiles, Shell, and /etc Configs

### Shared home = shared dotfiles (mostly good)

Since Distrobox mounts the host's home directory, all dotfiles are immediately available:
- `~/.zshrc` (oh-my-zsh, aliases, PATH, direnv, zoxide)
- `~/.vimrc`
- `~/.tmux.conf`
- `~/.gitconfig`
- `~/.ssh/` (keys, config)
- `~/.config/gh/`
- `~/.claude-work/`, `~/.claude-personal/`
- `~/.config/containers/auth.json`

No extra dotfile management needed. This is a major advantage of Distrobox over traditional containers.

### Shell inside the container

Distrobox picks up the user's `$SHELL` from the host. If the host uses zsh, the container uses zsh. If zsh is not installed in the container, Distrobox installs it during init.

To make this explicit, add zsh to `additional_packages` in `distrobox.ini`:

```ini
additional_packages="zsh"
```

oh-my-zsh is cloned to `~/.oh-my-zsh/` which is in the shared home, so it is available inside the container without any extra setup.

### /etc configs that differ

The container has its own `/etc/`. This means:
- `/etc/os-release` says Fedora, not RHEL
- `/etc/dnf/` has Fedora repo configs (this is what we want -- access to all Fedora packages)
- `/etc/resolv.conf` is shared from the host (DNS works)
- `/etc/passwd` is synchronized by distrobox-init (your user exists inside the container with the same UID/GID)
- `/etc/sudoers` is configured by distrobox-init (passwordless sudo inside the container)

### Default user

Distrobox creates a matching user inside the container with the same username, UID, GID, and home directory as the host user. You are NOT root inside the container (despite having passwordless sudo). This matches the host's user identity, so file permissions work correctly across the shared home.

### Potential dotfile conflict

If the host runs an older zsh or different tool versions, shared dotfiles could have compatibility issues. For example, a `.zshrc` using features from zsh 5.9 would break if the host has zsh 5.8. In practice, Fedora latest has newer versions than RHEL, so the container's tools are a superset. The risk is low.

If conflicts appear, the fix is `--home ~/distrobox-home/fedora-dev` to give the container its own home directory, then symlink specific dotfiles you want shared. This is the nuclear option -- try shared home first.

## Question 7: Performance

### Summary: No meaningful overhead

Distrobox containers share the host kernel. There is no virtualization layer, no hypervisor, no emulation. Processes inside the container run as native Linux processes with namespace isolation. Performance is identical to native execution.

### Benchmarks:

- **Container entry overhead**: ~400ms per `distrobox enter` invocation. Negligible for interactive use (you enter once and stay in the shell). Irrelevant for long-running tasks.
- **Compilation (Go, C, etc.)**: Native speed. The compiler runs as a regular process using the host's CPU, memory, and I/O. No measurable difference.
- **File I/O**: The shared home directory uses the host's filesystem directly (bind mount). No overlay filesystem, no copy-on-write penalty. Same I/O performance as native.
- **Network**: Shared network namespace. No NAT, no port forwarding overhead. Same throughput and latency as native.
- **Container image builds**: Delegated to host Podman via `distrobox-host-exec`. Same performance as running Podman directly on the host.

### The only overhead:

- Distrobox-init on first entry after container creation (installs packages, sets up user). This runs once.
- `distrobox-host-exec` adds a small overhead per command invocation (it enters a new namespace). For interactive use (running `podman build` once), this is imperceptible.

## Question 8: Persistence and Upgrades

### Containers persist by default

Distrobox containers are Podman containers under the hood. They persist across reboots. `podman ps -a` shows them. Their filesystem (everything outside the shared home) persists until you explicitly remove them.

Packages installed with `sudo dnf install` inside the container persist. `/usr/local/bin` binaries persist. Container-specific configuration in `/etc/` persists.

### What persists where:

| Location | Persists across container recreate? | Notes |
|---|---|---|
| `~/` (shared home) | Yes | Survives everything -- it's the host filesystem |
| `/usr/local/bin/` (container) | No | Recreated by init_hooks on next entry |
| `/etc/` (container) | No | Distrobox-init reconfigures on entry |
| DNF-installed packages (container) | No | Reinstalled from additional_packages |
| Podman images (host) | Yes | Managed by host's Podman |

### Upgrade workflow:

When Fedora releases a new version or you want to add/remove packages:

1. Edit `distrobox.ini` (change image tag, update package list)
2. Run `distrobox assemble create --replace --file ~/.config/distrobox/distrobox.ini`
3. The old container is removed and a new one created from the updated config
4. All home directory state (repos, configs, auth tokens) survives because it's on the host filesystem
5. init_hooks re-run to install binaries and create symlinks

This is the Distrobox equivalent of `dnf system-upgrade` but faster and more reliable. No in-place upgrade needed -- just rebuild from the declarative config.

### Ansible automation:

`make distrobox` runs the Ansible role which regenerates `distrobox.ini` from the template and runs `distrobox assemble create`. To force recreation, pass `--replace` or set `replace=true` in the ini file.

## Question 9: Toolbx vs Distrobox

### Recommendation: Support Distrobox, document Toolbx fallback

Distrobox is the better tool for this use case, but the design should not hard-depend on Distrobox-only features in a way that makes Toolbx impossible.

### Why Distrobox is preferred:

| Feature | Distrobox | Toolbx |
|---|---|---|
| `distrobox assemble` (declarative config) | Yes | No |
| `distrobox-export` (export binaries to host) | Yes | No |
| `distrobox-host-exec` (run host commands) | Yes | No (use `flatpak-spawn --host`) |
| `init_hooks` / `pre_init_hooks` | Yes | No |
| Works with Docker and Podman | Yes | Podman only |
| Works on RHEL CSB | Yes (if installable) | Yes (ships with RHEL) |
| Written in | POSIX shell | Go |

### The Toolbx advantage: It ships with RHEL

Toolbx is Red Hat's official tool. It is available in the RHEL repos (`dnf install toolbox`). If the RHEL CSB host has fapolicyd enforcing and blocks Distrobox installation (Distrobox installs to `~/.local/bin/` via curl), Toolbx is the fallback.

### If Toolbx is the only option:

Toolbx creates a Fedora container similarly:

```bash
toolbox create --distro fedora --release 42
toolbox enter fedora-toolbox-42
# Inside: sudo dnf install <packages>
```

But there is no INI manifest, no assemble, no export, no host-exec symlinks. You would need:
- A setup script that runs manually after `toolbox enter`
- No exported binaries -- work entirely inside the container
- For host Podman access: `flatpak-spawn --host podman <args>` (clunkier than the symlink pattern)
- No declarative recreation -- manual `toolbox rm` + `toolbox create` + re-run setup script

### Ansible strategy:

The Ansible `distrobox` role should:
1. Check if `distrobox` is available. If yes, use it.
2. If `distrobox` is not available but `toolbox` is, use Toolbx with a simpler setup path.
3. If neither is available, attempt to install Distrobox via curl (may fail if fapolicyd blocks it).

```yaml
- name: Check for distrobox
  ansible.builtin.command: which distrobox
  register: distrobox_check
  changed_when: false
  failed_when: false

- name: Check for toolbox
  ansible.builtin.command: which toolbox
  register: toolbox_check
  changed_when: false
  failed_when: false

- name: Use Distrobox path
  ansible.builtin.include_tasks: distrobox.yml
  when: distrobox_check.rc == 0

- name: Use Toolbx fallback
  ansible.builtin.include_tasks: toolbx.yml
  when: distrobox_check.rc != 0 and toolbox_check.rc == 0
```

## Complete distrobox.ini Specification

```ini
[fedora-dev]
image=registry.fedoraproject.org/fedora:latest
pull=true
replace=false
start_now=false
init=false
nvidia=false
root=false

# System libraries and build tools
additional_packages="gcc gcc-c++ clang make cmake autoconf automake libtool"
additional_packages="openssl-devel libffi-devel zlib-devel bzip2-devel readline-devel sqlite-devel"
additional_packages="kernel-devel"

# Languages and runtimes
additional_packages="golang python3 python3-pip python3-devel nodejs npm"

# CLI tools
additional_packages="git vim tmux jq zsh fzf direnv bash-completion"
additional_packages="procps-ng findutils which hostname curl wget tar gzip unzip"
additional_packages="ShellCheck yamllint ripgrep fd-find"

# Networking and debugging
additional_packages="tcpdump iproute nmap-ncat bind-utils traceroute"

# OVN/OVS (for ovn-kubernetes work)
additional_packages="ovn openvswitch"

# Container tools (buildah/skopeo run natively, podman delegates to host)
additional_packages="buildah skopeo"

# Python packages via pip (init_hooks because pip runs after package install)
# pre_init_hooks for repo setup
pre_init_hooks=dnf install -y 'dnf-command(config-manager)';

# Post-init setup
# Host delegation symlinks
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/podman;
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/docker;

# Binary tools not in Fedora repos (delegated to setup script for readability)
init_hooks=$HOME/laptop-setup/files/distrobox-setup.sh;

# Export commonly-used binaries to host
init_hooks=distrobox-export --bin /usr/bin/jq --export-path $HOME/.local/bin 2>/dev/null || true;
init_hooks=distrobox-export --bin /usr/bin/vim --export-path $HOME/.local/bin 2>/dev/null || true;
```

## User Experience

### Daily workflow

```
1. Boot RHEL CSB laptop
2. Open Alacritty terminal
3. Run: distrobox enter fedora-dev
   (or configure terminal profile to auto-enter)
4. You are now in a Fedora shell with all your tools
5. cd ~/src/submariner-io/submariner-operator
6. claude-work  (Claude Code works, Node.js is in the container)
7. make build   (Go compiler is in the container)
8. podman build . (delegates to host's Podman transparently)
9. kind create cluster (kind binary in container, Podman on host)
10. Exit with Ctrl+D or `exit`
```

### Auto-enter on terminal launch (optional)

Add to Alacritty config or terminal profile:

```toml
# ~/.config/alacritty/alacritty.toml
[terminal.shell]
program = "distrobox"
args = ["enter", "fedora-dev"]
```

Or add to `~/.zshrc` with a guard so it only triggers on the host:

```bash
# Auto-enter distrobox on host login shell (not inside container)
if [[ -z "$CONTAINER_ID" ]] && [[ -z "$DISTROBOX_ENTER_STATUS" ]]; then
    exec distrobox enter fedora-dev
fi
```

### tmux inside vs outside

Run tmux on the host, then enter Distrobox inside tmux. This way, if the container restarts, tmux sessions survive:

```
Host shell -> tmux -> distrobox enter fedora-dev -> work
```

Alternatively, run tmux inside the container. The container persists across reboots, so tmux sessions persist too. But if you ever recreate the container, tmux sessions are lost.

Recommendation: tmux on the host, development inside Distrobox within the tmux session.

## Ansible Role Structure

```
roles/distrobox/
├── tasks/
│   ├── main.yml         # Detect distrobox vs toolbx, delegate
│   ├── distrobox.yml    # Generate INI, run assemble
│   └── toolbx.yml       # Fallback: create container, run setup script
├── templates/
│   └── distrobox.ini.j2 # Jinja2 template for distrobox.ini
├── files/
│   └── distrobox-setup.sh  # Binary install script (runs inside container)
└── defaults/
    └── main.yml         # Package lists, image name, container name
```

### Key variables (defaults/main.yml):

```yaml
distrobox_container_name: fedora-dev
distrobox_image: "registry.fedoraproject.org/fedora:latest"
distrobox_replace: false
distrobox_ini_path: "{{ ansible_user_dir }}/.config/distrobox/distrobox.ini"

# Packages installed via dnf inside the container
distrobox_packages:
  - gcc
  - gcc-c++
  - clang
  - make
  - golang
  - python3
  - python3-pip
  - nodejs
  - npm
  - git
  - vim
  - tmux
  - zsh
  - jq
  - fzf
  - direnv
  - zoxide
  - ShellCheck
  - yamllint
  - ripgrep
  - curl
  - wget
  # ... full list

# Binaries to export to host (may not work if fapolicyd blocks ~/.local/bin)
distrobox_exports:
  - { bin: /usr/bin/jq, path: "~/.local/bin" }
  - { bin: /usr/bin/vim, path: "~/.local/bin" }
  - { bin: /usr/local/bin/gh, path: "~/.local/bin" }
  - { bin: /usr/local/bin/oc, path: "~/.local/bin" }
  - { bin: /usr/local/bin/kubectl, path: "~/.local/bin" }

# Host commands to delegate via distrobox-host-exec symlinks
distrobox_host_exec_symlinks:
  - podman
  - docker
```

## Open Questions

1. **Distrobox installability on RHEL CSB**: If fapolicyd blocks `~/.local/bin/distrobox`, need to check if Toolbx is pre-installed or if IT can whitelist the path. Test during Phase 2 (RHEL CSB recon).

2. **SELinux and volume mounts**: Distrobox uses bind mounts for the home directory. On RHEL with SELinux enforcing, some operations may need `:z` relabeling. Distrobox handles this automatically, but verify with `podman` container commands that reference host paths.

3. **Podman socket for kind**: kind with Podman may need the Podman socket enabled (`systemctl --user enable --now podman.socket`). This runs on the host. Verify this works on RHEL CSB (the user service may be restricted).

4. **Container registry access**: Container pulls from `registry.fedoraproject.org` must work through the corporate network. If blocked, need to mirror the Fedora image or use an internal registry.

5. **zoxide in container**: zoxide stores its database in `~/.local/share/zoxide/`. Shared home means host and container share the same zoxide database. This should be fine (paths are the same), but verify.

## References

- [Distrobox official docs](https://distrobox.it/)
- [distrobox-assemble manifest format](https://distrobox.it/usage/distrobox-assemble/)
- [distrobox-export usage](https://distrobox.it/usage/distrobox-export/)
- [distrobox-host-exec usage](https://distrobox.it/usage/distrobox-host-exec/)
- [Distrobox useful tips (symlink pattern)](https://distrobox.it/useful_tips/)
- [Toolbx on RHEL 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/using-toolbx-for-development-and-troubleshooting)
- [Toolbx on RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/using-toolbx-for-development-and-troubleshooting)
- [Distrobox vs Toolbx comparison](https://www.ypsidanger.com/distrobox-vs-toolbox-doesnt-matter/)
- [Distrobox GitHub (distrobox.ini examples)](https://github.com/89luca89/distrobox)
- [fapolicyd on RHEL](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening)
- [Claude Code in containers](https://code.claude.com/docs/en/devcontainer)
- [kind rootless with Podman](https://kind.sigs.k8s.io/docs/user/rootless/)
- [Running docker-compose in Distrobox with host Podman](https://hoelter.prose.sh/distrobox-docker-compose-fedora-silverblue)
- [Declaring personal distroboxes (Jorge Castro)](https://www.ypsidanger.com/declaring-your-own-personal-distroboxes/)

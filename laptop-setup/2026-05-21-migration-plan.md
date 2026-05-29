---
date: 2026-05-21T14:00:00-04:00
topic: laptop-setup
tags: [ansible, fedora, rhel-csb, macos, dotfiles, automation, i3, distrobox, tailscale, security]
---

# Workstation Setup Automation

Ansible project at `~/laptop-setup` to provision development machines (kept at `~/` not `~/src/` — needed before the repo layout exists). Uses Geerling's config override pattern: `default.config.yml` (shipped) + `config.yml` (gitignored overrides).

Two dimensions control what gets installed:
- **OS** (`os_family`: fedora/rhel/darwin/debian) — determines *how* (dnf vs brew vs apt, systemd vs launchd, paths)
- **Profile** (`work` or `personal`) — determines *what* (Red Hat tools, Vertex AI, downstream repos, registry auth)

## What It Sets Up

### Packages

**Core (all machines):**
- **CLI:** git, make, curl, tree, htop, tmux, jq, yq, watch, nmap, meld, tailscale, direnv, vim, zsh, fzf, zoxide, bitwarden-cli, keepassxc
- **Languages:** Go, Python 3, gcc
- **OpenShift/K8s:** oc, kubectl, kind, kustomize, helm, opm, subctl, openshift-install, gcloud, aws
- **Dev:** gh, gemini-cli, golangci-lint, gofumpt, govulncheck, gci, shellcheck, shfmt, yamllint, grype
- **K8s code-gen:** controller-gen, client-gen, informer-gen, lister-gen, deepcopy-gen, applyconfiguration-gen, defaulter-gen
- **Python (pip):** anthropic, pydantic, rpm-lockfile-prototype. Note: gitlint dropped (unmaintained 3+ years, supply chain risk).

**Work profile:**
- **Red Hat:** redhat-internal-cert-install, redhat-internal-openvpn-profiles, acli
- **Containers:** podman, buildah, skopeo. Docker optional (Fedora-only, rootless mode, NOT docker group)

**Linux desktop (any profile):**
- **OVN/OVS:** ovn-nbctl, ovn-sbctl, ovs-vsctl, ovn-trace, ovn-detrace
- **Networking:** tcpdump, bridge-utils, conntrack-tools, ethtool, iperf3, traceroute, iproute
- **Build:** clang (OVS/eBPF builds)
- **Desktop:** i3, i3status, i3lock, dmenu, Alacritty, nm-applet, scrot, feh, brightnessctl, gvim (vim-X11), dejavu-sans-mono-fonts, terminus-fonts

### Dotfiles

**All machines:** gitconfig (+ conditional include files: `~/.config/git/config-work`, `~/.config/git/config-personal`), vimrc, tmux.conf, ssh config, gh CLI configs, allowed signers file, pre-commit hook template
**Jinja2 templates:** zshrc (brew paths, AWS profile, claude-work/claude-personal aliases). Note: Vertex AI vars must NOT be in zshrc — they go in the alias/direnv only.
**Linux desktop:** zlogin, bashrc, htoprc, user-dirs.dirs, mimeapps.list, i3 config, i3status config, alacritty config
**macOS:** aerospace.toml (i3-like tiling WM), macOS defaults (keyboard repeat, smart quotes off, Finder, Dock)

Key custom settings to preserve:
- **zshrc**: oh-my-zsh (gallois theme, plugins: git/python/github/history-substring-search/gnu-utils/command-not-found), `EDITOR=vim`, `AWS_PROFILE=aws-acm-subm`, `eval "$(direnv hook zsh)"`, `eval "$(zoxide init zsh)"`, aliases (`gv`, `gi`, `ping`/`pingg`/`ping8`), claude-work/claude-personal aliases. Remove: Vertex AI exports (move to alias/direnv), stale `GOOGLE_CLOUD_PROJECT`, hardware-specific Bluetooth aliases.
- **gitconfig**: `logg` alias, meld difftool, `autocorrect=1`, `push.default=simple`, gh credential helper, Red Hat CA cert paths (work profile only, template conditional). `pushInsteadOf` to route pushes over SSH (YubiKey touch required) while pulls stay HTTPS (no touch). SSH commit signing enabled (`gpg.format=ssh`, `commit.gpgsign=true`) using a no-touch YubiKey key — Claude Code signs commits automatically (YubiKey plugged in, no tap), pushing still requires tap. Conditional includes for per-directory email: `includeIf "gitdir:~/src/openshift/"` → work email, `includeIf "gitdir:~/src/dfarrell07/"` → personal email. Prevents committing with wrong identity.
- **tmux.conf**: Ctrl+A prefix, vi-mode copy/paste.
- **vimrc**: 2-space hard tabs, 5000 history, restore cursor position, spell check, clipboard sharing.
- **ssh config**: `VisualHostKey=yes`, host entries for gh/gist (YubiKey identity), code.engineering/gitlab.cee (RH git key, work profile only). GitHub host gets `ControlMaster auto`, `ControlPath ~/.ssh/sockets/%r@%h-%p`, `ControlPersist 600` — one YubiKey tap opens a 10-minute multiplexed session for pushes.
- **i3 config**: Alt modifier, PulseAudio/PipeWire volume keys (pactl works via pipewire-pulseaudio compat), brightness keys, dmenu, nm-applet autostart, auto-lock on idle (i3lock via xautolock or xidlehook). Display setup script for external monitors (xrandr, hardware-specific — template with auto-detect or manual per-machine override).
- **gh CLI**: `co` alias for `pr checkout`, HTTPS protocol (ensures clones/pulls don't need YubiKey — gitconfig `pushInsteadOf` handles the SSH push gate separately).

### SSH Keys

From ansible-vault. Two YubiKey keys per device:
- **Auth key** (touch required): ed25519-sk with `-O verify-required`. Used for SSH login and git push.
- **Signing key** (no touch): ed25519-sk with `-O no-touch-required`. Used for git commit signing. YubiKey must be plugged in but no tap needed — Claude Code can sign commits automatically.

Both public keys uploaded to GitHub (auth key as Authentication Key, signing key as Signing Key). Red Hat git key encrypted in vault (work profile only). Allowed signers file at `~/.config/git/allowed_signers`.

### Git Repos (~80)

All repos under `~/src/` with URL-mirroring layout. GitHub repos at `<org>/<repo>`, internal repos prefixed by short host name:

```
~/src/
├── submariner-io/submariner-operator/    # github (default)
├── openshift/ovn-kubernetes/
├── dfarrell07/notes-ai/
├── code.engineering/dfarrell/            # code.engineering.redhat.com
└── gitlab.cee/dfarrell/                  # gitlab.cee.redhat.com
```

Defined in `repos.yml` with url, dest, category, enabled, optional extra_remotes. Cloned with `update: false`. Fork remotes added idempotently. VPN-dependent repos use `failed_when: false`. Filter by category: `-e repo_category=ovnk`.

Categories: personal, bpfman, downstream, konflux, ovnk, cncf.
Work profile default: all. Personal profile default: personal + ovnk + cncf.

### Third-Party RPM Repos (dnf-based OS only)

acli, docker-ce, expressvpn, gh-cli, google-chrome, google-cloud-sdk, rpmfusion (free + nonfree + nvidia + steam), slack, redhat. Each gated by a boolean with per-OS defaults.

### System (become: true, Linux only)

Firewall, SSH hardening, and Tailscale config detailed in the Security section.

- **CA certs**: 2022-IT-Root-CA.pem, Eng-CA.crt → `/etc/pki/ca-trust/source/anchors/` (work profile)
- **Kernel**: blacklist intel_vbtn
- **Lid close**: `HandleLidSwitch=ignore`, `HandleLidSwitchExternalPower=ignore` in `/etc/systemd/logind.conf` — no suspend on lid close
- **dnf-automatic**: enable `dnf5-automatic.timer`, config: `apply_updates=yes`, `upgrade_type=default`
- **Services**: fail2ban, libvirtd, cups, dkms, expressvpn (gated). Docker disabled by default (start on demand)
- **Virtualization**: libvirt, qemu-kvm

### User Environment (become: false)

- **Shell**: zsh default, oh-my-zsh (git clone to `~/.oh-my-zsh`, not a package — idempotent check for existing dir)
- **Claude Code**: install on all machines. Two isolated instances on work laptop (see below). Remote Control server (personal profile). Does NOT manage `~/.claude/` configs directly — handled by instance isolation.

**Claude Code instance isolation (work laptop):**

Two separate instances to prevent auth/data leakage between work (Vertex AI) and personal (Anthropic account):

```bash
# Never set Vertex vars in .zshrc — they override everything and can't be turned off per-project
# CLAUDE_CODE_USE_VERTEX checks for presence, not value — setting to 0 doesn't work

# Work instance
alias claude-work='CLAUDE_CONFIG_DIR=~/.claude-work \
  CLAUDE_CODE_USE_VERTEX=1 \
  ANTHROPIC_VERTEX_PROJECT_ID=itpc-gcp-hcm-pe-eng-claude \
  CLOUD_ML_REGION=global \
  claude'

# Personal instance
alias claude-personal='CLAUDE_CONFIG_DIR=~/.claude-personal claude'
```

Each instance gets isolated: settings, credentials, session history, MCP servers, plugins. Authenticate each separately (`/login` or API key). Use direnv `.envrc` per project for automatic switching — explicitly `unset` the other context's vars. Never run both in the same working directory.

**Claude Code global config (both instances):**

`~/.claude-work/CLAUDE.md` and `~/.claude-personal/CLAUDE.md`:
- Always use `--signoff` (`-s`) on git commits
- You cannot push — pushes route over SSH and require YubiKey touch. Commit to a branch, then ask the user to push. Once pushed, open the PR.
- Prefer terse responses.
- After any manual install, config tweak, or system change — update the corresponding Ansible role in `~/laptop-setup` to capture it. The Ansible repo is the source of truth for machine state.

`~/.claude-work/settings.json`:
- Permissions: do NOT use `allow: *`. Add specific allow rules for common operations. Add deny rules for sensitive file reads (`~/.ssh/**`, `~/.aws/**`, `~/.kube/**`, `**/.env`, `~/.vault_pass`). YubiKey push gate is the safety net for outbound changes, but file reads need explicit protection.
- Move machine-specific config (model, MCP servers) to `settings.local.json` so `settings.json` can be shared/versioned.
- Plugins: work instance gets shipyard, jira, release-management, claude-skills. Personal instance gets claude-skills only. Install via `claude plugin add` with marketplace config in `settings.json` — the `claude` role templates `settings.json` with the `extraKnownMarketplaces` and `enabledPlugins` blocks per instance.
- MCP servers: work instance gets Atlassian MCP. Personal instance gets none or different set.
- Sandbox: enable with `bubblewrap` (`dnf install bubblewrap socat`). Set `"sandbox": {"enabled": true}` in settings.

- **Container registry auth**: tokens from vault via credential helper (docker-credential-secretservice), deployed to `~/.config/containers/auth.json` with mode 0600 (work profile)
- **Distrobox**: Fedora dev container — safety valve for RHEL CSB (work profile on RHEL). See Distrobox section below for full spec.

## Security

### Secrets Management

- **Ansible Vault**: AES-256 encryption. Password in `~/.vault_pass` (mode 0600, outside repo, never committed).
- **Vault conventions**: all secret vars use `vault_` prefix, referenced from plaintext vars files (`ssh_key: "{{ vault_ssh_key }}"`). `no_log: true` on every task that handles vault-decrypted secrets.
- **Vault rotation**: `ansible-vault rekey` when credentials are compromised or periodically. Procedure documented in repo.
- **Templates**: never contain raw secrets — reference vault variables only. Dotfiles with interpolated secrets deployed mode 0600. Note: Vertex AI vars are NOT in zshrc — they live in direnv `.envrc` and claude-work alias only.
- **Git hygiene**: `.gitignore` covers `config.yml`, `.vault_pass*`, `*.vault_pass`. Pre-commit hook greps for `password:`, `token:`, `BEGIN OPENSSH` — deployed by the dotfiles role as a git template hook (`git config --global init.templateDir ~/.config/git/template`, hook in `~/.config/git/template/hooks/pre-commit`).
- **Allowed signers**: `~/.config/git/allowed_signers` deployed by dotfiles role — maps email to signing public key for `git log --show-signature` verification.

### Network

- **Firewall**: `public` zone, not FedoraWorkstation (which opens 1025-65535). Default-DROP policy. Allow only SSH (non-default port) + specific dev ports as needed. Allow essential ICMP (destination-unreachable, time-exceeded, echo-reply) — full stealth breaks Path MTU Discovery.
- **Tailscale**: mesh VPN for machine-to-machine access. No open ports on public interfaces. WireGuard underneath — cryptographically invisible to port scanners.
- **VPN isolation**: never run ExpressVPN and Red Hat VPN simultaneously — routing conflicts leak traffic between contexts. Verify routes with `ip route` after connecting. Check DNS leaks with `resolvectl status`.

### SSH

- **Server hardening** (`/etc/ssh/sshd_config`): `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`, `LoginGraceTime 30`, `AllowUsers <username>`, `X11Forwarding no`, `AllowAgentForwarding no`, `AllowTcpForwarding no`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`.
- **Non-default port**: noise reduction (eliminates automated bot traffic), not a security control. Keep below 1024 (privileged) to prevent local attackers from binding if sshd is down.
- **fail2ban**: installed and enabled. Reduces log noise, catches misconfigured scanners.
- **YubiKey FIDO2**: YubiKey 5C NFC with firmware 5.7+ (pre-5.7 keys vulnerable to EUCLEAK CVE-2024-45678). Ed25519-sk keys with `-O verify-required` (PIN + touch). Private key never leaves hardware. 100 resident key slots. Non-resident keys preferred for fixed workstations (attacker needs both YubiKey AND key handle file on disk).
- **Backup YubiKey**: separate key enrolled on a second YubiKey, stored securely. Prevents lockout if primary is lost/damaged.
- **Key permissions**: `~/.ssh/` mode 0700, private keys 0600, public keys 0644.
- **GNOME Keyring conflict**: GNOME Keyring's SSH agent does not support FIDO2 keys — breaks signing and auth. Disable by removing `/etc/xdg/autostart/gnome-keyring-ssh.desktop` (or `mkdir -p ~/.config/autostart && cp /etc/xdg/autostart/gnome-keyring-ssh.desktop ~/.config/autostart/ && echo Hidden=true >> ~/.config/autostart/gnome-keyring-ssh.desktop`). Use OpenSSH's ssh-agent instead. Also: `-O no-touch-required` flag is not preserved when importing resident keys via `ssh-keygen -K` — keep original key handle files.
- **Port knocking**: dropped. Unencrypted, replay-vulnerable. Tailscale replaces this — no open port to hide.

### Containers

- **Docker group**: never join. Docker group membership = passwordless root (can mount host filesystem, read `/etc/shadow`). Docker daemon disabled by default, started on demand only.
- **Podman preferred**: rootless, daemonless, fork-exec model. 11 kernel capabilities vs Docker's 14.
- **Docker rootless mode**: if Docker specifically needed, run rootless. No group membership required.
- **Registry auth**: credential helper (`docker-credential-secretservice` on Linux, keychain on Mac), not base64 in config files. Deployed to `~/.config/containers/auth.json` (persistent, survives reboot) with mode 0600. Set `REGISTRY_AUTH_FILE` env var.

### Cloud CLI Credentials

All store tokens in plaintext on disk by default:
- `gcloud`: `~/.config/gcloud/credentials.db`, `application_default_credentials.json`
- `aws`: `~/.aws/credentials`
- `gh`: `~/.config/gh/hosts.yml` (plaintext fallback when keyring unavailable)
- `acli`: `~/.config/` config files

Mitigations:
- Directory permissions mode 0700, credential files mode 0600.
- Prefer short-lived tokens: `gcloud auth application-default login` (ADC), `aws configure sso`.
- `gh`: verify keyring is working (`gh auth status`), install `gnome-keyring` as backend.
- Full-disk encryption (LUKS on Linux, FileVault on Mac) — protects all credential files at rest. RHEL CSB ships with LUKS. For Fedora and macOS, enable during OS install (not automated by this project — must be done at install time).

### Phone / Mobile Security

- No SSH key on phone. No direct network path to laptop.
- GitHub Issues as task queue: phone can create issues but cannot execute anything on laptop directly.
- If phone is extracted (Cellebrite, border search): attacker gets GitHub token, can create issues. Mitigated by `--allowedTools` whitelist on laptop's task processor + PRs require human review.
- Strong lockscreen PIN (not biometric alone — biometric can be compelled in some jurisdictions).

### Travel

- Shut down laptop (not sleep) when crossing borders — LUKS keys are in memory during sleep.
- Remove phone from Tailscale network before travel, re-add after.
- Consider: travel with clean phone (factory reset), no GitHub auth, reconnect after clearing border.
- `authorized_keys`: remove any non-YubiKey entries before travel.
- Review and revoke unnecessary cloud CLI sessions (`gcloud auth revoke`, `gh auth logout`).

### Claude Code Security

**Instance isolation:**
- Never set `CLAUDE_CODE_USE_VERTEX` globally — checks presence, not value. Must be completely unset for personal instance.
- Separate `CLAUDE_CONFIG_DIR` prevents credential, MCP server, and session history leakage between contexts.
- Known bugs: session cross-contamination in same directory (#27658), branch swapping (#60295). Never run both instances in the same working directory.
- `~/.claude.json` is shared regardless of config dir — low-impact but avoid simultaneous runs when possible.

**Repo trust (cloned repos can attack Claude Code):**
- Malicious CLAUDE.md files inject into Claude's system prompt — can instruct arbitrary file reads and exfiltration.
- Malicious `.mcp.json` starts attacker-controlled processes with full user privileges at Claude Code startup.
- Malicious `.claude/settings.json` can enable `enableAllProjectMcpServers` to auto-approve MCP servers (TrustFall, unpatched).
- Malicious `.envrc` (direnv) can redirect `ANTHROPIC_BASE_URL` to exfiltrate API keys.
- Git `core.fsmonitor` in crafted repos executes code when Claude runs `git status`.

**Mitigations (deploy via Ansible):**
- Add file-read deny rules: `Read(~/.ssh/**)`, `Read(~/.aws/**)`, `Read(~/.kube/**)`, `Read(**/.env)`, `Read(~/.vault_pass)`
- `git config --global core.hooksPath ~/.config/git/hooks` — override per-repo hooks
- `git config --global core.fsmonitor false` — prevent fsmonitor code execution
- Inspect CLAUDE.md, .mcp.json, .claude/settings.json, .envrc before running Claude in any new repo
- Consider enabling Claude Code sandbox (`bubblewrap`): `"sandbox": {"enabled": true}` in settings

**Supply chain incidents (2026):**
- Claude Code npm: source leak + concurrent axios trojan (March 2026). Pin exact npm versions.
- Bitwarden CLI npm: 90-minute trojan in `@bitwarden/cli@2026.4.0` (April 2026). Verify on 2026.4.1+.
- Multiple MCP server CVEs (Inspector, Filesystem, Git). Audit MCP servers before enabling.

### Ansible Playbook Security

- `become: false` is the default in `ansible.cfg`. Only the system play and specific tasks escalate.
- No play-level `become: true` on the user play — individual tasks only when genuinely needed.
- Sudo password from vault, not hardcoded.
- All module names use FQCN (`ansible.builtin.copy`, not `copy`).
- All `shell:`/`command:` tasks have `changed_when:` and idempotency guards (`creates:`, `when:`).
- `requirements.yml` pins all external collection versions to prevent supply chain drift.

## macOS-Specific

Items that only apply to macOS (personal Mac desktop):

**Packages (brew cask):** alacritty, google-chrome, aerospace (i3-like tiling WM), bitwarden, keepassxc, slack, docker (if needed)

**YubiKey FIDO2 fix:** macOS built-in OpenSSH lacks FIDO2 support. Must install `brew install openssh libfido2 ssh-askpass`. Ensure Homebrew's ssh is first in PATH. Disable Apple's launchd ssh-agent (doesn't support FIDO2) — use Homebrew's ssh-agent instead.

**Window management:** AeroSpace (`brew install --cask nikitabobko/tap/aerospace`) — i3-inspired tiling WM, TOML config, no SIP disable needed. Config at `~/.config/aerospace/aerospace.toml`.

**macOS defaults:** deploy via `community.general.osx_defaults` — key repeat enabled, smart quotes/dashes off, Finder shows extensions and full path, screenshots to ~/Downloads, screen lock immediately on sleep.

**Firewall:** `socketfilterfw --setglobalstate on`, stealth mode on. Works per-application, not per-port like firewalld.

**Services (launchd):** systemd units become launchd plists in `~/Library/LaunchAgents/`. Needed for: Claude Code Remote Control server, task queue poller, brew autoupdate.

**Brew autoupdate:** `brew tap domt4/autoupdate && brew autoupdate start 43200 --upgrade --cleanup` — equivalent of dnf-automatic.

**Registry credential helper:** `docker-credential-osxkeychain` instead of `docker-credential-secretservice`.

**Manual (cannot automate):** TCC permissions — Accessibility (AeroSpace), Full Disk Access (terminal), App Management (brew autoupdate). Requires GUI interaction.

## Design Decisions

- **Ansible** — one tool for packages, dotfiles, services, secrets
- **Distrobox** — run Fedora tools on RHEL CSB without root
- **Alacritty** — replaces urxvt. GPU-accelerated, TOML config, Dracula theme, in Fedora repos
- **Bitwarden** — replaces LastPass (actively exploited stolen vaults, $438M+ losses). Open source, self-hostable via Vaultwarden, YubiKey FIDO2 on free tier, CLI for automation.
- **KeePassXC** — offline vault for highest-value secrets (recovery codes, vault password, backup keys). Local-only, never touches cloud. YubiKey challenge-response.
- **YubiKey 5C NFC (firmware 5.7+)** — already have two (purchased Nov 2025, likely 5.7+, verify with `ykman info`). 100 resident key slots, native Ed25519, USB-C + NFC for Pixel. Pre-5.7 keys are vulnerable to EUCLEAK CVE-2024-45678.
- **Rejected**: chezmoi (marginal over ansible-vault), mise (CVE-2026-35533, not in repos), GNU Stow (no templating), yadm (no advantage), Nix (no RHEL support), Devbox (needs /nix), LastPass (breached, actively exploited), 1Password (closed source)

## Project Structure

```
~/laptop-setup/
├── ansible.cfg                     # become: false default, vault password path
├── Makefile
├── site.yml                        # two plays: system (become: true) + user (become: false)
├── requirements.yml                # pinned collection versions (community.general, containers.podman)
├── default.config.yml              # profile: work, os-specific defaults
├── config.yml                      # gitignored overrides (set profile, toggle features)
├── .ansible-lint                   # profile: production, FQCN required
├── .yamllint
├── inventory/localhost.yml
├── group_vars/all/
│   ├── repos.yml                   # ~80 git repos
│   └── vault.yml                   # ansible-vault encrypted: SSH keys, registry auth, env vars
└── roles/
    ├── common/                     # OS detection, profile setup, dirs (~/.ssh/sockets, ~/.config/git/template/hooks, etc.), prerequisites
    ├── repos_dnf/                  # third-party RPM repos (dnf-based OS only)
    ├── packages/                   # brew/dnf/apt, pip, go, binary downloads. CLI tool versions pinned in variables (oc_version, kubectl_version, etc.) — update by changing the variable, re-run playbook
    ├── dotfiles/                   # config files (copy + template), OS-conditional
    ├── ssh/                        # SSH keys from vault
    ├── git_repos/                  # clone repos, add fork remotes, profile filters
    ├── redhat/                     # CA certs, VPN packages (work profile)
    ├── containers/                 # podman, docker, registry auth (work profile)
    ├── virtualization/             # libvirt, qemu-kvm (Linux only)
    ├── cloud_tools/                # OpenShift/K8s/cloud CLI binaries
    ├── desktop/                    # i3, Alacritty (Linux desktop), zsh (all)
    ├── system/                     # firewall, kernel modules, services (Linux only)
    ├── distrobox/                  # Distrobox + Fedora container (RHEL CSB)
    └── claude/                     # Claude Code install, instance isolation (work/personal), Remote Control (personal), task queue poller (systemd timer + script)
```

## Makefile Targets

| Target | Sudo? (Linux) | What it does |
|---|---|---|
| `make all` | yes | Full setup |
| `make dotfiles` | no | Deploy config files |
| `make packages` | yes | Install packages |
| `make repos` | no | Clone all git repos |
| `make repos-ovnk` | no | OVN-K repos only |
| `make repos-konflux` | no | Konflux repos only |
| `make repos-personal` | no | Personal repos only |
| `make ssh` | no | Deploy SSH keys |
| `make desktop` | no | i3 + shell + apps |
| `make system` | yes | Firewall, kernel, services |
| `make redhat` | yes | CA certs and VPN |
| `make cloud` | yes | Cloud CLIs |
| `make distrobox` | no | Distrobox + Fedora container |

Utilities: `make check` (dry run), `make diff` (dotfile diffs), `make vault-edit`, `make lint` (ansible-lint + yamllint).

## Bootstrap (fresh machine)

**Fedora/RHEL:** `sudo dnf install ansible-core git` → clone repo → `ansible-galaxy install -r requirements.yml` → `make all`
**macOS:** `xcode-select --install` → install Homebrew → `brew install ansible git openssh libfido2` → clone repo → `ansible-galaxy install -r requirements.yml` → `make all`. Note: must install Homebrew's OpenSSH (macOS built-in doesn't support FIDO2 YubiKey keys). Verify `which ssh` shows `/opt/homebrew/bin/ssh` after setup.

Vault password bootstrap: install Bitwarden CLI first (`brew install bitwarden-cli` or `dnf install bitwarden-cli`), `bw login`, retrieve vault password, write to `~/.vault_pass`. Then `make all` can decrypt everything else.

## Implementation Phases

1. Skeleton + dotfiles → `make dotfiles` works
2. **RHEL CSB recon** → `make check` on CSB, identify fapolicyd/sudo/USBGuard constraints early. Determines whether Distrobox is optional or required.
3. Packages + third-party repos → `make packages` idempotent
4. Git repos with category filtering → `make repos-ovnk` works
5. Vault + SSH + Red Hat + containers → `ssh -T git@github.com` works
6. System + remaining roles → `make all` idempotent
7. Profile support → `make all` with `profile: personal` skips work-only roles
8. Task queue repo → create private repo, laptop-side issue poller + `claude -p` runner
9. Multi-OS → test on macOS, adjust conditionals
10. Distrobox dev container → if CSB blocks host tools, full dev environment inside Fedora container

**Testing:**
- **Fedora VM** (libvirt): spin up a fresh Fedora VM, run `make all`, verify idempotency (second run = 0 changed). Fastest feedback loop — test before touching real hardware.
- **macOS**: run `make all` on the Mac desktop with `profile: personal`. Verify skipped roles, brew installs, dotfiles.
- **RHEL CSB**: `make check` (dry run) first, then `make all` with graceful failure handling. Document what breaks.
- **Idempotency**: every test runs the playbook twice. Second run must report 0 changed.
- **Linting**: `make lint` passes (ansible-lint + yamllint) before any real machine run.
- **Smoke tests**: after `make all` — `ssh -T git@github.com`, `claude-work --version`, `claude-personal --version`, `oc version`, `podman info`, `gh auth status`.

## Remote Access

### Work Machines (Vertex AI)

Remote Control, Dispatch, cloud sessions, and Routines all require claude.ai OAuth — they do not work with Vertex AI auth. The only remote access to Claude Code on work machines is SSH.

**From another computer (Tailscale + SSH + tmux):**
- Tailscale mesh VPN connects machines — no open ports, NAT traversal automatic
- `ssh work-machine` → `tmux attach -t claude`
- Claude Code runs in a tmux session with full local access (repos, MCP servers, skills, plugins)
- Vertex AI billing and auth preserved
- Requires: machine on + Tailscale connected + YubiKey physically present on the connecting machine
- Open question: does Red Hat IT allow Tailscale on RHEL CSB? If not, fallback is SSH over Red Hat VPN (OpenConnect)

**From phone (GitHub Issues → PR workflow — no SSH to laptop):**

Phone never has direct access to the laptop. GitHub is the only communication channel.

Flow:
1. Phone: create GitHub Issue with spec/plan in a private repo (GitHub app, voice dictation)
2. Laptop: polls for open issues (`gh issue list`), Claude Code picks one up, creates branch, does the work, opens PR linking the issue
3. Phone: review PR in GitHub app (diffs, inline comments, approve, merge)
4. Claude Code comments on the issue with progress/questions, closes issue when PR merged

Phone side:
- GitHub mobile app — native issue creation, PR review, voice dictation via keyboard
- No Termux/Tasker needed for basic workflow
- Optional: work or personal Claude app to help compose issue specs (GitHub connector read-only)

Laptop side:
- Systemd timer (every 2 min, oneshot prevents overlap) — see Task Queue Repo section for full design
- Safety: `--allowedTools` whitelist per repo, `--max-turns 50`, `--max-budget-usd 5.00`, 30 min timeout

Security:
- No SSH key on phone, no Tailscale needed
- Phone only needs GitHub auth — device extraction gets GitHub token but no laptop shell access
- Attacker can create issues but `--allowedTools` limits what Claude can do
- PRs require human review before merge — no direct commits to main
- Full audit trail via issues, PRs, and comments

### Personal Machines (Anthropic account)

Personal Anthropic Pro account. All remote features available.

- **Remote Control**: `claude remote-control` as persistent server. Steer from claude.ai/code or Claude mobile app. No inbound ports, survives network drops.
- **Dispatch**: pair Claude mobile app with Claude Desktop. Send tasks from phone, machine runs them.
- **Cloud sessions**: claude.ai/code → remote session against GitHub repos, no local machine needed. Teleport to local later.
- **Routines**: scheduled or API-triggered cloud runs.
- **Claude Desktop MCP (Mac)**: full local MCP server support + 50+ built-in connectors (GitHub, Slack, Jira, Google Drive, etc.)

### Claude Chat Apps (separate from Claude Code)

Two apps, different accounts, different capabilities:

**Work app** (managed, Red Hat email):
- Claude chat under Team/Enterprise plan, admin-enforced restrictions (e.g. Gmail/GCal blocked)
- Check which connectors IT has enabled — Jira and GitHub useful for reading issues/repos from phone
- Cannot push commits or connect to Claude Code on laptop — separate product, separate auth

**Personal app** (Pro/Max account):
- Full 50+ connectors, voice mode, no restrictions
- Mobile: remote MCP only (publicly reachable servers), GitHub read-only (can't push)
- Desktop: full local MCP + remote MCP

Neither app replaces the git task queue for sending work to Claude Code.

### Cross-Machine Work

- GitHub is the bridge — push from one, pull/clone on the other
- Cloud sessions (personal account) can work on any GitHub repo
- SSH to work machine for anything that needs Vertex billing or local-only repos

### Known Limitations

- **Work phone → Claude Code is async only** — git task queue, 2-3 min delay. Hard constraint of Vertex AI auth.
- **YubiKey required physically** for SSH — no key = no access. Keep backup YubiKey accessible.
- **Personal features need Pro/Max account** — have Pro, all features available.
- **Work Claude app connectors** are admin-managed — availability depends on Red Hat IT policy.

## RHEL CSB Constraints

RHEL CSB (Corporate Standard Build) is Red Hat's internal hardened workstation image. Exact hardening profile unknown publicly, but if STIG-based these restrictions apply:

**Likely blocked without IT exception:**
- **Third-party repos** (Tailscale, ExpressVPN, Docker CE) — STIG prohibits non-Red Hat repos including EPEL
- **Homebrew/Linuxbrew** — installs outside RPM trust database, blocked by fapolicyd if enforcing
- **pip --user, go install, npm global** — binaries in ~/  paths blocked by fapolicyd (deny-all, permit-by-exception for RPM-trusted paths)
- **Custom firewall rules** — STIG requires drop zone, admin-managed
- **Kernel module changes** — `/etc/modprobe.d/` is root-owned, may require `module.sig_enforce=1`
- **Docker** — not in RHEL repos since RHEL 8, third-party repo required
- **systemd service enable/disable** — requires root, STIG mandates specific services

**Likely works:**
- **Podman rootless** — ships with RHEL, Red Hat supported. Needs one-time admin setup of `/etc/subuid` and `/etc/subgid`
- **Distrobox** — installs to `~/.local/bin/`, uses rootless Podman. Potential blocker: fapolicyd may block scripts in `~/.local/bin/`. Red Hat's official alternative is Toolbx.
- **SSH server** — STIG allows but heavily restricts (key-only, restricted ciphers, logging)
- **SELinux** — enforcing with targeted policy, mandatory. Affects container volume mounts (use `:z` flag)
- **LUKS encryption** — CSB ships with full-disk encryption

**Uncertain (needs IT verification):**
- **Sudo access** — may be scoped to specific commands, not blanket `ALL`
- **USBGuard** — STIG requires it, blocks unknown USB devices. YubiKeys may or may not be whitelisted by default. Adding devices requires root access to `/etc/usbguard/rules.conf`.
- **fapolicyd** — if enforcing, breaks most developer toolchains (even Red Hat's own Ansible Automation Platform is "not supported when fapolicyd is enforcing"). This is the single biggest risk to the plan.
- **Tailscale** — requires third-party repo + systemd service. Fallback: SSH over Red Hat VPN (OpenConnect).

**Impact on the plan:**
- Distrobox becomes critical, not optional — most dev tools may need to run inside a Fedora container
- The `make system` and `make packages` targets may partially fail on CSB — need graceful handling
- Two-tier approach: minimal host (Podman, Distrobox/Toolbx, SSH, tmux) + full dev env inside container

**Distrobox dev container spec:**

Host provides only: Podman rootless, Distrobox (or Toolbx fallback), SSH, tmux. All dev tools live in a Fedora container.

Container setup via `distrobox.ini` (Ansible generates from Jinja2 template, runs `distrobox assemble create`):
- Base: Fedora latest
- Packages via `additional_packages`: Go, Python 3, gcc, clang, vim, zsh, jq, tmux, direnv, fzf, zoxide, shellcheck, shfmt, yamllint, Node.js
- Binary downloads via `init_hooks`: oc, kubectl, kind, kustomize, helm, opm, subctl, gh, golangci-lint, grype, yq, Claude Code
- Container commands (podman, buildah, skopeo) delegate to host via `distrobox-host-exec` symlinks — no nested containers
- kind works: binary in container calls host's Podman to create cluster nodes

Shared with host (Distrobox default): home directory, display, network. All dotfiles, SSH keys, Claude Code config, oh-my-zsh work automatically.

Primary workflow: `distrobox enter dev` → work inside the container. Export sparingly to host (`distrobox-export --bin` for jq, gh, oc) — fapolicyd may block exports anyway.

Upgrade: edit `distrobox.ini`, run `distrobox assemble create --replace`. Home-dir state survives.

See `laptop-setup/2026-05-28-distrobox-dev-environment.md` for full design.

## Task Queue Repo

Private GitHub repo for phone-to-laptop async task communication via Issues and PRs.

- Private repo, single user access only.
- Phone GitHub token scoped to this repo only (fine-grained PAT) — limits blast radius if phone is compromised.

**Issue format:**
```
repo: submariner-operator
<spec/plan in body>
```
`repo:` line maps to local paths via a config file (e.g. `submariner-operator` → `~/src/submariner-io/submariner-operator`).

**State machine via labels:** `queued` → `processing` → `done` / `failed`. Failed issues stay open with an error comment (exit code, stderr, branch name). Retry: remove `failed`, add `queued`.

**Poller:** systemd timer (every 2 min after completion, `Type=oneshot` prevents overlap). Processes issues sequentially, oldest first. Branch naming: `claude/<issue-number>-<slug>`. Opens PR linking the issue, comments with progress.

**Safety:** `--max-turns 50`, `--max-budget-usd 5.00`, `timeout 1800` (30 min). Systemd adds `MemoryMax=4G`, `CPUQuota=80%`.

**Critical: task queue must run in a container** — `Bash` in allowedTools means Claude can read any file, exfiltrate via DNS/git notes/PR comments, and push to any repo. Mitigations:
- Run `claude -p` in a disposable Podman container with only the target repo mounted
- No user home directory, SSH keys, or cloud credentials inside container
- `--network=none` or network namespace allowing only HTTPS to github.com
- Repo-scoped deploy key as sole credential (not user's gh auth)
- Separate PAT for poller (reads issues) vs executor (pushes code) — executor cannot write issues/comments
- Destroy container after each task

**PRs require human review/merge** — no unreviewed code lands on main.

See `laptop-setup/2026-05-28-claude-task-queue-design.md` for full implementation design.

## Manual Setup

**All machines:**
- YubiKey SSH key enrollment (verify firmware 5.7+ with `ykman info`)
- Bitwarden setup + migrate from LastPass
- KeePassXC setup for offline vault (recovery codes, vault password)
- `gh auth login`
- Chrome sign-in (bookmarks/extensions)
- ExpressVPN activation

**Work profile:**
- `oc login --web` to Konflux cluster (`kflux-prd-rh02`)
- `podman login registry.redhat.io` (also brew.registry, stage.registry)
- Red Hat entitlements in `/etc/pki/entitlement/*.pem` — needed for RPM lockfile updates. RHEL CSB should have these pre-provisioned; Fedora needs `subscription-manager`.
- `gcloud auth login` + `gcloud auth application-default login`
- `acli` login (8 configs)
- Kerberos ticket (`kinit`)
- Docker/Podman registry login refresh (when tokens expire)

**Personal profile:**
- Claude Code Remote Control server setup
- Claude Desktop app + mobile pairing for Dispatch

## References

**Ansible workstation patterns:**
- [geerlingguy/mac-dev-playbook](https://github.com/geerlingguy/mac-dev-playbook) — gold standard Ansible workstation setup
- [jsm84/fedora-ansible](https://github.com/jsm84/fedora-ansible) — Fedora as RHEL CSB replacement

**Claude Code:**
- [Claude Code on Vertex AI](https://code.claude.com/docs/en/google-vertex-ai) — auth, env vars, limitations
- [Claude Code Remote Control](https://code.claude.com/docs/en/remote-control) — requires claude.ai OAuth, not Vertex
- [Claude Code headless mode](https://code.claude.com/docs/en/headless) — `claude -p` for automation

**Security:**
- [Tailscale](https://tailscale.com/) — mesh VPN, WireGuard-based
- [Distrobox](https://distrobox.it/) — rootless container dev environments
- [DISA STIG for RHEL](https://www.stigviewer.com/stig/red_hat_enterprise_linux_9/) — hardening requirements
- [fwknop](https://github.com/mrash/fwknop) — Single Packet Authorization (evaluated, not adopted — Tailscale used instead)

**Mobile/phone integration:**
- [GitHub mobile app](https://github.com/mobile) — issue creation, PR review
- [Termux](https://termux.dev/) — Linux terminal on Android (optional, for advanced workflows)
- [Claude Code on Android](https://github.com/ferrumclaudepilgrim/claude-code-android) — proot-Ubuntu approach (future option)

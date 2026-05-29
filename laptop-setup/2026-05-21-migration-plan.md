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
- **CLI:** git, make, curl, tree, htop, tmux, mosh, jq, yq, watch, nmap, meld, tailscale, direnv, vim, zsh, fzf, zoxide, bitwarden-cli, keepassxc
- **Languages:** Go, Python 3, gcc
- **OpenShift/K8s:** oc, kubectl, kind, kustomize, helm, opm, subctl, openshift-install, gcloud, aws
- **Dev:** gh, golangci-lint, gofumpt, govulncheck, gci, shellcheck, shfmt, yamllint, grype (verify ≥v0.104.1, CVE-2025-65965 credential leak in earlier versions), ansible-lint
- **K8s code-gen:** controller-gen, client-gen, informer-gen, lister-gen, deepcopy-gen, applyconfiguration-gen, defaulter-gen
- **Python (pip):** anthropic, pydantic, rpm-lockfile-prototype. Note: gitlint dropped (unmaintained 3+ years, supply chain risk).

**Work profile:**
- **Red Hat:** redhat-internal-cert-install, redhat-internal-openvpn-profiles, acli
- **Containers:** podman, buildah, skopeo. Docker optional (Fedora-only, rootless mode, NOT docker group)

**Linux desktop (any profile):**
- **OVN/OVS:** ovn-nbctl, ovn-sbctl, ovs-vsctl, ovn-trace, ovn-detrace
- **Networking:** tcpdump, bridge-utils, conntrack-tools, ethtool, iperf3, traceroute, iproute
- **Build:** clang (OVS/eBPF builds)
- **Desktop:** i3, i3status, i3lock, dmenu, Alacritty, nm-applet, scrot, feh, brightnessctl, gvim (vim-X11), dejavu-sans-mono-fonts, terminus-fonts, xss-lock or xidlehook (screen lock timer — xautolock is unmaintained, dropped)
- **Security:** yubikey-manager (ykman), bubblewrap, socat (Claude Code sandbox), docker-credential-secretservice

### Dotfiles

**All machines:** gitconfig (+ conditional include files: `~/.config/git/config-work`, `~/.config/git/config-personal`), vimrc, tmux.conf, ssh config, gh CLI configs, allowed signers file, pre-commit hook template
**Jinja2 templates:** zshrc (brew paths, AWS profile, claude-work/claude-personal aliases). Note: Vertex AI vars must NOT be in zshrc — they go in the alias/direnv only.
**Linux desktop:** zlogin, bashrc, htoprc, user-dirs.dirs, mimeapps.list, i3 config, i3status config, alacritty config
**macOS:** aerospace.toml (i3-like tiling WM), macOS defaults (keyboard repeat, smart quotes off, Finder, Dock)

Key custom settings to preserve:
- **zshrc**: oh-my-zsh (gallois theme, plugins: git/python/history-substring-search/gnu-utils/command-not-found), `EDITOR=vim`, `AWS_PROFILE=aws-acm-subm`, `eval "$(direnv hook zsh)"`, `eval "$(zoxide init zsh)"`, `zstyle ':omz:update' mode auto`. Remove: `github` plugin, Vertex AI exports, stale `GOOGLE_CLOUD_PROJECT`, hardware-specific Bluetooth aliases.

  **Shell aliases and functions:**
  ```bash
  # Claude Code in tmux (always — sessions survive disconnects)
  cw() {  # Claude Work — creates/attaches tmux session named after current dir
    local s="work-${PWD##*/}"
    tmux new-session -As "$s" "CLAUDE_CONFIG_DIR=~/.claude-work \
      CLAUDE_CODE_USE_VERTEX=1 ANTHROPIC_VERTEX_PROJECT_ID=itpc-gcp-hcm-pe-eng-claude \
      CLOUD_ML_REGION=global claude; zsh"
  }
  ccp() {  # Claude Code Personal (not `cp` — shadows copy command)
    local s="personal-${PWD##*/}"
    tmux new-session -As "$s" "CLAUDE_CONFIG_DIR=~/.claude-personal claude; zsh"
  }
  cls() { tmux list-sessions 2>/dev/null | grep -E "work-|personal-"; }
  ca() { tmux attach -t "$(tmux list-sessions -F '#{session_name}' | grep -E 'work-|personal-' | head -1)"; }
  cstatus() {  # What Claude is doing across all sessions
    for s in $(tmux list-sessions -F '#{session_name}' 2>/dev/null | grep -E 'work-|personal-'); do
      echo "=== $s ==="; tmux capture-pane -t "$s" -p | tail -3; echo; done
  }

  # Device access (Tailscale + mosh)
  alias laptop='mosh laptop.tail'
  alias mac='mosh mac-desktop.tail'
  laptop-claude() { mosh laptop.tail -- tmux attach -t "${1:-work}"; }
  mac-claude() { mosh mac-desktop.tail -- tmux attach -t "${1:-personal}"; }

  # Remote Control (personal Mac)
  alias rc='claude --remote-control'

  # Git push (YubiKey awareness)
  gpush() { echo "YubiKey touch required..."; git push "$@"; }

  # Quick nav (supplement zoxide)
  alias src='cd ~/src'
  alias subm='cd ~/src/submariner-io'
  alias ovnk='cd ~/src/openshift/ovn-kubernetes'
  alias notes='cd ~/src/dfarrell07/notes-ai'

  # Task queue (create issue from CLI)
  cq() { gh issue create --repo dfarrell07/claude-tasks --title "$1" --body "$2"; }

  # Status
  alias yk='ykman info'                    # YubiKey status
  alias vpn='ip route | grep -E "tun|wg"'  # VPN status
  alias ts='tailscale status'              # Tailscale mesh status

  # Existing aliases (keep)
  alias gv="gvim -vp"
  alias gi="grep -rniI --color=always --exclude-dir=vendor"
  alias ping="ping -i .2"
  alias pingg="ping www.google.com"
  alias ping8="ping 8.8.8.8"
  ```
- **gitconfig**: `logg` alias, meld difftool, `autocorrect=1`, `push.default=simple`, gh credential helper, Red Hat CA cert paths (work profile only, template conditional). `pushInsteadOf` to route pushes over SSH (YubiKey touch required) while pulls stay HTTPS (no touch). SSH commit signing enabled (`gpg.format=ssh`, `commit.gpgsign=true`) using a no-touch YubiKey key — Claude Code signs commits automatically (YubiKey plugged in, no tap), pushing still requires tap. Conditional includes for per-directory email: `includeIf "gitdir:~/src/openshift/"` → work email, `includeIf "gitdir:~/src/dfarrell07/"` → personal email. Prevents committing with wrong identity.
- **tmux.conf**: Ctrl+A prefix, vi-mode copy/paste, `monitor-activity on`, `visual-activity on` (alerts when background Claude sessions produce output).
- **zlogin**: `xset s off`, `xset -dpms` (prevent display sleep). Remove: `xrdb ~/.Xdefaults` (Xdefaults dropped), RVM loading (dropped), commented-out bluetooth/inotify.
- **bashrc**: PATH setup (`~/.local/bin`, `~/bin`). Remove: stale `GOOGLE_CLOUD_PROJECT`.
- **vimrc**: 2-space hard tabs, 5000 history, restore cursor position, spell check, clipboard sharing.
- **ssh config**: `VisualHostKey=yes`, host entries for gh/gist (YubiKey identity), code.engineering/gitlab.cee (RH git key, work profile only). GitHub host gets `ControlMaster auto`, `ControlPath ~/.ssh/sockets/%r@%h-%p`, `ControlPersist 600` — one YubiKey tap opens a 10-minute multiplexed session for pushes.
- **i3 config**: Alt modifier, PulseAudio/PipeWire volume keys (pactl works via pipewire-pulseaudio compat), brightness keys, dmenu, nm-applet autostart, auto-lock on idle (i3lock via xss-lock or xidlehook). Display setup script for external monitors (xrandr, hardware-specific — template with auto-detect or manual per-machine override).
- **gh CLI**: `co` alias for `pr checkout`, HTTPS protocol (ensures clones/pulls don't need YubiKey — gitconfig `pushInsteadOf` handles the SSH push gate separately). Note: current config says `git_protocol: ssh` — must change to `https` during setup.
- **gh hosts.yml**: `github.com`, user `dfarrell07`, `git_protocol: https`.

### SSH Keys

From ansible-vault. Two YubiKey keys per device:
- **Auth key** (touch required): ed25519-sk with `-O verify-required`. Used for SSH login and git push.
- **Signing key** (no touch): ed25519-sk with `-O no-touch-required`. Used for git commit signing. YubiKey must be plugged in but no tap needed — Claude Code can sign commits automatically.

Both public keys uploaded to GitHub (auth key as Authentication Key, signing key as Signing Key). Red Hat git key encrypted in vault (work profile only). Allowed signers file at `~/.config/git/allowed_signers`.

### Git Repos (~114)

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

Categories: personal, bpfman, downstream, konflux, ovnk, cncf, misc.
Work profile default: all (except disabled). Personal profile default: personal + ovnk + cncf.
Disabled (do not clone): dependent-issues, ipset-go, Huberman-Lab-Podcast-Transcripts.
Fork remotes: naming convention `dfarrell_<shortname>` (e.g. `dfarrell_op`, `dfarrell_lh`, `dfarrell_subm`).

Full repo list must be populated in `repos.yml` during implementation phase 4. Currently 114 repos across: submariner-io (14), stolostron (8), konflux (14), ovnk (37), openshift (misc, 4), cncf (7), personal (8), bpfman (6), misc/disabled (16).

### Third-Party RPM Repos (dnf-based OS only)

acli, docker-ce, mullvad/protonvpn, gh-cli, google-chrome, google-cloud-sdk, rpmfusion (free + nonfree + nvidia + steam), slack, redhat. Each gated by a boolean with per-OS defaults.

### System (become: true, Linux only)

Firewall, SSH hardening, and Tailscale config detailed in the Security section.

- **CA certs**: 2022-IT-Root-CA.pem, Eng-CA.crt → `/etc/pki/ca-trust/source/anchors/` (work profile)
- **auditd rules**: monitor reads of `~/.ssh/`, `~/.aws/`, `~/.gnupg/`, `~/.kube/config`, `~/.vault_pass`, `~/.claude/.credentials.json`. Deploy to `/etc/audit/rules.d/claude-code.rules`.
- **Kernel**: blacklist intel_vbtn
- **Lid close**: `HandleLidSwitch=ignore`, `HandleLidSwitchExternalPower=ignore` in `/etc/systemd/logind.conf` — no suspend on lid close
- **dnf-automatic**: enable `dnf5-automatic.timer`, config: `apply_updates=yes`, `upgrade_type=default`
- **Services**: fail2ban, libvirtd, cups, dkms, mullvad/protonvpn (gated). Docker disabled by default (start on demand)
- **Virtualization**: libvirt, qemu-kvm

### User Environment (become: false)

- **Shell**: zsh default, oh-my-zsh (git clone to `~/.oh-my-zsh`, not a package — idempotent check for existing dir)
- **Claude Code**: install on all machines. Two isolated instances on work laptop (see below). Remote Control server (personal profile). Does NOT manage `~/.claude/` configs directly — handled by instance isolation.

**Claude Code instance isolation (work laptop):**

Two separate instances to prevent auth/data leakage between work (Vertex AI) and personal (Anthropic account). Use the `cw`/`ccp` shell aliases (defined in zshrc section) which wrap Claude in tmux with the correct env vars per instance. Never set Vertex vars in `.zshrc` globally — `CLAUDE_CODE_USE_VERTEX` checks for presence, not value. Use direnv `.envrc` per project for auto-switching — explicitly `unset` the other context's vars. Never run both in the same working directory.

**Claude Code global config (both instances):**

**Data privacy:**
- Vertex AI (work): prompts go to Google infrastructure, not Anthropic. No training. Customer-controlled retention. Telemetry to Anthropic off by default.
- Anthropic Pro (personal): turn off "Improve Claude" in privacy settings (5-year retention if on, 30 days if off). No ZDR or DPA on Pro plan. Consumer terms, not commercial.
- Disable non-essential telemetry: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` in both instances.
- Cross-contamination: running personal instance while cd'd into a work repo sends work code through Anthropic. Wrapper scripts should verify current directory is appropriate for the instance before launching.

`~/.claude-work/CLAUDE.md` and `~/.claude-personal/CLAUDE.md` (keep under 150 lines each):
- Always use `--signoff` (`-s`) on git commits
- You cannot push — pushes route over SSH and require YubiKey touch. Commit to a branch, then ask the user to push. Once pushed, open the PR.
- Prefer terse responses.
- After any manual install, config tweak, or system change — update the corresponding Ansible role in `~/laptop-setup` to capture it. The Ansible repo is the source of truth for machine state.

**Parallel work (worktrees):**
- Always use `claude --worktree <name>` or the `cw`/`ccp` aliases (which create tmux sessions) when running multiple Claude sessions on the same repo. Never run two sessions in the same working directory — causes branch swapping (#60295).
- Worktrees live in `.claude/worktrees/<name>/` (add to `.gitignore`).
- Remote Control server supports `--spawn worktree` for multi-device parallel sessions.
- Agent Teams (experimental, `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) for coordinated multi-agent work.

**Context hierarchy:**
- **Managed**: `/etc/claude-code/CLAUDE.md` — organization-wide (CSB only)
- **User**: `~/.claude-work/CLAUDE.md` or `~/.claude-personal/CLAUDE.md` — personal preferences, always loaded
- **Project**: `./CLAUDE.md` or `./.claude/CLAUDE.md` — team-shared, checked into git
- **Local**: `./CLAUDE.local.md` — personal project overrides, gitignored
- **Nested**: `src/api/CLAUDE.md` — monorepo/subdirectory rules, loaded on-demand when files in that path are read
- **Path-scoped rules**: `.claude/rules/*.md` with `paths:` frontmatter for conditional loading
- **Skills**: `.claude/skills/*.md` — loaded only when invoked or auto-matched by description
- All CLAUDE.md files concatenate (not override). Keep total under ~200 lines across all levels.

**Memory**: auto-memory at `~/.claude/projects/<project>/memory/`. First 200 lines of MEMORY.md loaded at session start. Topic files loaded on-demand. Not shared across machines — project-specific learnings, not portable config.

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

- **Ansible Vault**: AES-256 encryption. Tiered vault IDs with different trust levels:
  - **Critical** (SSH keys, production creds): password derived from YubiKey HMAC-SHA1 challenge-response (`ykchalresp -2 "memorized-pin"`). Exists in NO password manager. Both YubiKeys configured with same HMAC secret. Paper backup of hex secret in fireproof safe.
  - **Infra** (registry tokens, API keys): password in KeePassXC (offline `.kdbx` on USB drive, master password + YubiKey).
  - **Dev** (env vars, non-sensitive config): password in Bitwarden (cloud, acceptable blast radius).
  - Password files at `~/.vault_pass_critical`, `~/.vault_pass_infra`, `~/.vault_pass_dev` (all mode 0600, outside repo).
- **Vault conventions**: all secret vars use `vault_` prefix, referenced from plaintext vars files (`ssh_key: "{{ vault_ssh_key }}"`). `no_log: true` on every task that handles vault-decrypted secrets.
- **Vault rotation**: `ansible-vault rekey` when credentials are compromised or periodically. Procedure documented in repo.
- **Templates**: never contain raw secrets — reference vault variables only. Dotfiles with interpolated secrets deployed mode 0600. Note: Vertex AI vars are NOT in zshrc — they live in direnv `.envrc` and claude-work alias only.
- **Git hygiene**: `.gitignore` covers `config.yml`, `.vault_pass*`, `*.vault_pass`. Pre-commit hook greps for `password:`, `token:`, `BEGIN OPENSSH` — deployed by the dotfiles role as a git template hook (`git config --global init.templateDir ~/.config/git/template`, hook in `~/.config/git/template/hooks/pre-commit`).
- **Allowed signers**: `~/.config/git/allowed_signers` deployed by dotfiles role — maps email to signing public key for `git log --show-signature` verification.

### Network

- **Firewall**: `public` zone, not FedoraWorkstation (which opens 1025-65535). Default-DROP policy. Allow only SSH (non-default port) + specific dev ports as needed. Allow essential ICMP (destination-unreachable, time-exceeded, echo-reply) — full stealth breaks Path MTU Discovery.
- **Tailscale**: mesh VPN for machine-to-machine access. No open ports on public interfaces. WireGuard underneath — cryptographically invisible to port scanners. Enable **Tailnet Lock** (free on Personal plan) — prevents rogue node injection even if Tailscale's coordination server is compromised. DERP relays are end-to-end encrypted (they forward WireGuard ciphertext, never hold private keys). Headscale (self-hosted) not needed for 2-3 devices — Tailnet Lock provides equivalent security without infrastructure burden.
- **VPN isolation**: never run personal VPN (Mullvad/ProtonVPN) and Red Hat VPN simultaneously — routing conflicts leak traffic between contexts. Verify routes with `ip route` after connecting. Check DNS leaks with `resolvectl status`.

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
- If phone is extracted (Cellebrite, border search): attacker gets GitHub token, can create issues. Mitigated by `--allowedTools` whitelist on laptop's task processor + PRs require human review + container isolation.
- Strong lockscreen PIN (not biometric alone — biometric can be compelled in some jurisdictions).
- **Pixel hardening**: enable Advanced Protection Mode (Android 16, one toggle — hardware USB blocking, auto-reboot 72h, blocks sideloading). Use Work Profile via Shelter (F-Droid) for app isolation (GitHub, Claude apps in work profile). Lockdown Mode (Power+Volume Up) before border crossings — disables biometrics, forces PIN. Set USB default to "No data transfer."
- **GrapheneOS** (optional, for frequent travelers): 18h auto-reboot (vs 72h stock), duress PIN (silent factory reset), stronger USB blocking. Compatible with GitHub Mobile, Claude app via Sandboxed Google Play.

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
- `requirements.yml` pins all external collection versions to prevent supply chain drift. Note: `ansible-galaxy install` has no integrity verification by default — run `ansible-galaxy collection verify` after install.
- **Ansible repo integrity**: commit signing enforced via branch protection. Consider `ansible-sign` for GPG project-level verification before `make all` runs on a fresh machine.
- **Audit logging**: Claude Code logs full JSONL transcripts to `~/.claude/projects/`. Enable OpenTelemetry for structured monitoring: `CLAUDE_CODE_ENABLE_TELEMETRY=1`, `OTEL_LOGS_EXPORTER=console`, `OTEL_LOG_TOOL_DETAILS=1`. Add auditd rules for sensitive file reads (`~/.ssh/`, `~/.aws/`, `~/.kube/`, `~/.vault_pass`).

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

## Chrome Hardening

Deploy via Ansible as policy JSON — works without enterprise enrollment on Linux (`/etc/opt/chrome/policies/managed/`) and macOS (`/Library/Managed Preferences/`).

- **Extension allowlist**: block all extensions by default (`ExtensionInstallBlocklist: ["*"]`), allowlist by Chrome Web Store ID (uBlock Origin, Bitwarden, etc.)
- **Disable built-in password manager**: `PasswordManagerEnabled: false`, `AutofillCreditCardEnabled: false`, `AutofillAddressEnabled: false` — use Bitwarden instead
- **Disable sync on work profile**: `SyncDisabled: true` — prevents syncing compromised extensions across devices
- **HTTPS-only**: `HttpsOnlyMode: "force_enabled"`
- **Site isolation**: `SitePerProcess: true` (prevents Chrome from relaxing it under memory pressure)
- **Separate profiles**: work profile (Jira, GitHub, GCP) and personal profile. Different extension sets, separate cookie jars.

A malicious extension with `<all_urls>` + `cookies` permissions can steal session tokens for Jira, GitHub, and GCP Console. The extension allowlist is the primary defense.

## Design Decisions

- **Ansible** — one tool for packages, dotfiles, services, secrets
- **Distrobox** — run Fedora tools on RHEL CSB without root
- **Alacritty** — replaces urxvt. GPU-accelerated, TOML config, Dracula theme, in Fedora repos
- **Bitwarden** — replaces LastPass (actively exploited stolen vaults, $438M+ losses). Open source, self-hostable via Vaultwarden, YubiKey FIDO2 on free tier, CLI for automation.
- **KeePassXC** — offline vault for highest-value secrets (recovery codes, vault password, backup keys). Local-only, never touches cloud. YubiKey challenge-response.
- **YubiKey 5C NFC (firmware 5.7+)** — already have two (purchased Nov 2025, likely 5.7+, verify with `ykman info`). 100 resident key slots, native Ed25519, USB-C + NFC for Pixel. Pre-5.7 keys are vulnerable to EUCLEAK CVE-2024-45678.
- **Mullvad or ProtonVPN** — replaces ExpressVPN. ExpressVPN's parent Kape Technologies has adware origins (Crossrider), went fully private in 2023 (zero public oversight), employed a UAE offensive hacker as CIO. Mullvad: no email signup, police-raid-tested, WireGuard-only. ProtonVPN: open source clients, Swiss jurisdiction. Either is a trust upgrade.
- **Proprietary tools (accepted)**: Claude Code (source-available, Anthropic Commercial ToS — no open alternative with equivalent capability), acli (Appfire EULA — Atlassian MCP server used as complement, but acli still needed for some workflows), gcloud CLI (source-available, no public repo — required for GCP)
- **Rejected**: chezmoi (marginal over ansible-vault), mise (CVE-2026-35533, not in repos), GNU Stow (no templating), yadm (no advantage), Nix (no RHEL support), Devbox (needs /nix), LastPass (breached, actively exploited), 1Password (closed source), ExpressVPN (Kape ownership, ex-adware, closed source client, went private), gemini-cli (Google killing free API June 2026)

## Architecture: Ansible + Claude Code Skill

Ansible handles deterministic provisioning (packages, dotfiles, services, repos). A Claude Code skill wraps it to handle what Ansible can't — interactive auth, diagnostic review, post-run verification, and iterative fixing.

**Skill invocation:** `/laptop-setup` or `make all`

**Skill phases:**
1. **Pre-flight** — Claude checks: OS detection, YubiKey plugged in, vault passwords accessible, network connectivity, existing state (fresh vs re-run)
2. **Ansible execution** — runs `ansible-playbook site.yml` with appropriate tags/profile, streams output
3. **Output review** — Claude reviews Ansible output for failures, diagnoses root causes, suggests or applies fixes
4. **Interactive auth** — guides user through manual auth steps that Ansible can't automate: `gh auth login`, `oc login --web`, `gcloud auth login`, `acli` login, Bitwarden setup
5. **Post-run verification** — runs smoke tests (`ssh -T git@github.com`, `claude-work --version`, `podman info`, etc.), reports results
6. **Summary** — what succeeded, what failed, what needs manual attention, what to run next

**Failure handling:** if a role fails, Claude reads the error, checks docs/known issues, suggests the fix. User approves, Claude re-runs just the failed role with `--tags`. Iterates until clean or surfaces unresolvable issues.

**File structure:**
```
~/laptop-setup/
├── CLAUDE.md                       # Project-level instructions for Claude Code
├── skills/
│   └── setup/SKILL.md              # Main setup skill (pre-flight → ansible → review → auth → verify)
├── .claude/
│   ├── SKILLS.md                   # Skill index
│   └── rules/                      # Path-scoped rules for role-specific guidance
├── scripts/
│   ├── preflight.sh                # Pre-flight checks (YubiKey, vault, network)
│   ├── smoke-test.sh               # Post-run verification
│   └── auth-guide.sh               # Interactive auth step guidance
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
│   ├── repos.yml                   # ~114 git repos
│   └── vault.yml                   # ansible-vault encrypted: SSH keys, registry auth, env vars
└── roles/
    ├── common/                     # OS detection, profile setup, dirs (~/.ssh/sockets, ~/.config/git/template/hooks, etc.), prerequisites
    ├── repos_dnf/                  # third-party RPM repos (dnf-based OS only)
    ├── packages/                   # install methods by source:
    │   #   dnf: system packages, gh (gh-cli repo), acli (acli repo), gcloud (google repo)
    │   #   brew: python3, node (+ mac-only: all CLI tools). gemini-cli dropped (Google killing free API June 2026)
    │   #   binary download (/usr/local/bin): oc, kubectl, kind, kustomize, opm, openshift-install, grype, yq, helm
    │   #   go install: controller-gen, gofumpt, govulncheck, gci, *-gen, golangci-lint
    │   #   pip --user: anthropic, pydantic, rpm-lockfile-prototype
    │   #   npm (via brew node): Claude Code
    │   #   curl to ~/.local/bin: subctl, distrobox
    │   #   dnf (new): tailscale, direnv, zoxide, fzf, bitwarden-cli, keepassxc, ansible-lint, xss-lock
    │   #   versions pinned in variables — update by changing variable, re-run
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

Utilities: `make check` (dry run), `make diff` (dotfile diffs), `make vault-edit`, `make lint` (ansible-lint + yamllint), `make smoke-test` (post-run verification).

Claude Code skill: `/laptop-setup` — runs pre-flight → `make all` → reviews output → guides interactive auth → runs smoke tests → reports summary. Use this instead of bare `make all` for a guided, diagnostic setup experience.

## Bootstrap (fresh machine)

**Fedora/RHEL:**
```bash
sudo dnf install ansible-core git ykpers  # ykpers provides ykchalresp
git clone https://github.com/dfarrell07/laptop-setup ~/laptop-setup
cd ~/laptop-setup
ansible-galaxy collection install -r requirements.yml
# Get vault passwords (see below)
make all
```

**macOS:**
```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install ansible git openssh libfido2 ykman
git clone https://github.com/dfarrell07/laptop-setup ~/laptop-setup
cd ~/laptop-setup
ansible-galaxy collection install -r requirements.yml
# Get vault passwords (see below)
make all
```
Note: must install Homebrew's OpenSSH (macOS built-in doesn't support FIDO2). Verify `which ssh` shows `/opt/homebrew/bin/ssh` after setup.

**Vault password retrieval** (before `make all`):
1. Plug in YubiKey → `ykchalresp -2 "your-pin"` → write to `~/.vault_pass_critical`
2. Mount KeePassXC USB drive → retrieve infra password → write to `~/.vault_pass_infra`
3. Install Bitwarden (official binary, NOT npm) → `bw login` → retrieve dev password → write to `~/.vault_pass_dev`
4. `make all` decrypts each vault tier. Shred temp files after.

**Chicken-and-egg note:** `ykpers` (Fedora) / `ykman` (brew) must be in the bootstrap install line because vault passwords are needed before the playbook can decrypt anything. `gh auth login` happens after `make all` installs gh — the git_repos role handles cloning, which needs gh auth first. Run `make packages && make dotfiles && gh auth login && make repos` if running incrementally.

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

### iPad Pro (thin client + Claude app)

- **Primary role**: portable thin client for steering Claude Code remotely + Claude app for voice/connectors
- **Remote Control**: `claude.ai/code` in Safari → steer Mac desktop sessions. Pro plan, full features.
- **SSH**: Blink Shell + Tailscale → mosh/tmux to work laptop. YubiKey USB-C works for SSH auth.
- **Claude app**: work account (managed, Jira/GitHub connectors) + personal account (full connectors, voice mode)
- **GitHub app**: issue creation, PR review (same as phone but better with keyboard + larger screen)
- **Dispatch**: send tasks from Claude iOS app to Mac Desktop app
- **No local Claude Code** — iPad runs iPadOS, not practical for Node.js

### Pixelbook (lightweight dev + thin client)

- **Primary role**: thin client via browser + light local dev in Crostini
- **Remote Control**: `claude.ai/code` in Chrome → steer Mac desktop sessions. Best thin-client experience.
- **SSH**: Crostini terminal + Tailscale (Android app, Baguette networking) → mosh/tmux to work laptop
- **Local Claude Code**: install in Crostini (`npm install -g @anthropic-ai/claude-code`) with Anthropic Pro for light personal work. 8GB RAM limits heavy use.
- **YubiKey limitation**: FIDO2 SSH does NOT work in Crostini (only FIDO1/U2F). Use GPG/PIV workaround or Tailscale SSH (avoids key management).
- **Browser-based dev**: GitHub Codespaces, code-server for VS Code workflows
- **Keep ChromeOS** — full Linux install not worth the trade-offs on 8GB/2017 hardware

### Tailscale Mesh

All keyboard devices on Tailscale for SSH access. Phone stays off (GitHub Issues only).

| Device | Tailscale | SSH to laptop | SSH to Mac | Remote Control |
|---|---|---|---|---|
| Linux laptop | Yes | — | Yes | No (Vertex) |
| Mac desktop | Yes | Yes | — | Server |
| iPad Pro | Yes | Yes (Blink) | Yes (Blink) | Client (Safari) |
| Pixelbook | Yes (Android app) | Yes (Crostini) | Yes (Crostini) | Client (Chrome) |
| Pixel phone | No | No | No | No (GitHub Issues) |

Tailnet Lock enabled. Phone excluded to minimize attack surface (no SSH key, no Tailscale creds).

### Cross-Device Patterns

**Core principles:**
- **tmux everywhere** — every Claude Code session that might outlast your attention starts in tmux. Decouples the session from any specific device or connection.
- **mosh over SSH for mobile** — iPad and Pixelbook use mosh (UDP, survives sleep/network switches). Install `mosh-server` on laptop and Mac. Blink Shell supports mosh natively.
- **GitHub as durable output** — PRs, issues, review comments are the right place for Claude's output when you need cross-device visibility. Push notifications reach every device.
- **Phone is for capture, not execution** — dictate specs, create issues, receive notifications. Don't code on it.
- **Mac desktop is the always-on anchor** — overnight automation (cron), Remote Control server, Dispatch target, fallback when laptop is asleep.

**Workflow scenarios:**
- **Couch/iPad**: Blink Shell → mosh to laptop → tmux attach → `/review` a PR. Detach when done, review continues.
- **Coffee shop/Pixelbook**: Local Claude Code in Crostini for spec writing. Remote Control via Chrome to Mac for implementation.
- **Walking/phone**: Voice → Claude app → format as GitHub Issue → Dispatch to Mac or pickup later.
- **Desk/laptop**: Multiple tmux windows, parallel Claude sessions (feature + review + tests).
- **Traveling/iPad only**: mosh to laptop for debugging. Remote Control to Mac as fallback. Claude app for triage.
- **Device handoff**: tmux is the universal session persistence. SSH/mosh from new device, `tmux attach`.
- **Overnight**: Cron on Mac desktop runs nightly reviews, posts results as PR comments. GitHub notifications reach phone.

**Notifications:** Claude output → GitHub (PRs, comments) → push notifications to phone/iPad. tmux `monitor-activity` for terminal-based alerts. Slack webhooks for non-GitHub automation.

**Session persistence:**
- iPad sleep + mosh → auto-resumes (best option)
- iPad sleep + SSH → reconnect, `tmux attach`
- Pixelbook sleep → Crostini survives, reattach tmux locally
- Mac desktop → always running, never sleeps

### Hardware Options (portable coding rigs)

**Current gear:** ZSA Moonlander (custom cables, daily driver), Shokz (sport/safety).

**AR glasses:**
- **XREAL 1S** ($449 + Hub $49 + Rx ~$75 = ~$575) — wearable USB-C monitor. 1920x1200/eye (16:10), X1 chip (3DoF), 700 nits, electrochromic dimming, 82g. Terminal-readable at 14pt+ font. 2-4 hour sessions. **Security: no mic, no camera, no wireless radio** — physically can't capture or transmit data. Chinese company (Beijing) but hardware constraints make origin irrelevant. Do NOT install Nebula app on work devices. Use as plain USB-C display.
- **Even Realities G2** (~$1,050 with ring + Rx) — notification/approval HUD only (25 chars x 7 lines, can't review code). Tap ring to approve, voice via Whisper. **SECURITY: DO NOT USE FOR WORK.** 4-mic always-on array, Chinese company subject to National Intelligence Law, undisclosed cloud audio processing. No hardware mic kill switch. Can capture room conversations including work calls. Personal-only, and even then only when no work context is present.

**Keyboards:**
- **Moonlander** (already owned) — works with iPad/phone via USB-C direct (recognized as USB HID). Disable RGB LEDs to avoid iPad power complaints. Tethered but functional. Primary desk keyboard.
- **Glove80** ($399, recommended upgrade) — wireless Bluetooth 5.0, 80 keys (more than Moonlander), contoured key wells, ZMK, 5 BT device profiles. Pairs with laptop, iPad, phone, Pixelbook simultaneously. Best "upgrade from Moonlander" — more keys (easier transition), better ergonomics (sculpted wells), wireless. Not ultra-portable but fits in a backpack.
- **Wireless Corne** (~$200 from Typeractive) — 42-key, BLE, 200g, pocket-sized. Travel-only keyboard. Steep adaptation from Moonlander (30 fewer keys). Best for coffee shop / park bench.
- **Logitech MX Keys Mini** ($100) — airplane keyboard. Not ergonomic but fits on tray tables.

**Voice:**
- **Shokz** (already owned) — fine for voice, outdoor mic quality limited by wind
- **Bose Ultra Open** (~$300, upgrade) — best open-ear audio quality for listening to Claude responses + good mic
- **Shokz OpenComm UC** (~$160, upgrade) — boom mic version, better voice input than sport Shokz
- Claude mobile app voice mode: intent-level instructions ("fix the retry logic"), not syntax. No terminal access.

**The rigs:**
- **Park bench**: iPad Pro + Moonlander (USB-C) + XREAL 1S. ~1.5kg. All compute remote.
- **Coffee shop**: iPad Pro + Corne (BLE) or Glove80 (BLE). XREAL optional.
- **Walking**: Shokz/Bose in ears + phone in pocket. Claude app voice for thinking, GitHub notifications for agent status. G2 only if purely personal context (no work discussions nearby).
- **Airplane**: iPad Pro + MX Keys Mini or Magic Keyboard. XREAL awkward with seatmates.
- **Everywhere**: mosh + tmux to laptop/Mac over Tailscale. Device in hand is just a terminal.

### Known Limitations

- **Work phone → Claude Code is async only** — git task queue, 2-3 min delay. Hard constraint of Vertex AI auth.
- **YubiKey required physically** for SSH — no key = no access. Keep backup YubiKey accessible.
- **Personal features need Pro/Max account** — have Pro, all features available.
- **Work Claude app connectors** are admin-managed — availability depends on Red Hat IT policy.

## RHEL CSB Constraints

RHEL CSB (Corporate Standard Build) is Red Hat's internal hardened workstation image. Exact hardening profile unknown publicly, but if STIG-based these restrictions apply:

**Likely blocked without IT exception:**
- **Third-party repos** (Tailscale, Mullvad/ProtonVPN, Docker CE) — STIG prohibits non-Red Hat repos including EPEL
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
- Phone GitHub token: fine-grained PAT, Issues:Read/Write only, scoped to this single repo. No Contents permission. 30-day expiry. Never edit PATs via GitHub UI (bug silently reverts scope to all repos) — delete and recreate instead.

**Issue format:**
```
repo: submariner-operator
<spec/plan in body>
```
`repo:` line maps to local paths via a config file (e.g. `submariner-operator` → `~/src/submariner-io/submariner-operator`).

**State machine via labels:** `queued` → `processing` → `done` / `failed`. Failed issues stay open with an error comment (exit code, stderr, branch name). Retry: remove `failed`, add `queued`.

**Poller:** systemd timer (every 2 min after completion, `Type=oneshot` prevents overlap). Processes issues sequentially, oldest first. Branch naming: `claude/<issue-number>-<slug>`. Opens PR linking the issue, comments with progress.

**Safety:** `--max-turns 50`, `--max-budget-usd 5.00`, `timeout 1800` (30 min). Systemd adds `MemoryMax=4G`, `CPUQuota=80%`.

**Critical: task queue must run in a bare Podman container, NOT Distrobox.** Distrobox provides zero security isolation (shares home, network, `/run/host` exposes entire host filesystem). It is an integration tool, not an isolation tool.

The task queue executor uses bare Podman with strict isolation:
```
podman run --rm --network=slirp4netns --read-only \
  --tmpfs /tmp:size=100m --tmpfs /home/claude:size=50m \
  --memory=4g --cpus=2 --pids-limit=256 \
  --cap-drop=ALL --security-opt=no-new-privileges \
  --userns=keep-id -v "$REPO:/workspace:Z" \
  -e ANTHROPIC_API_KEY="$KEY" claude-task-runner \
  -p "$PROMPT" --allowedTools "$TOOLS" --max-turns 50
```
- Only the target repo mounted (`:Z` for SELinux private label) — no home dir, no SSH keys, no cloud creds
- Network: `slirp4netns` (Podman default for rootless). Primary defense is filesystem isolation, not network — container has no credentials to exfiltrate. PR/issue comments remain a channel regardless of network policy.
- Repo-scoped deploy key as sole credential (not user's gh auth)
- Separate PAT for poller (reads issues) vs executor (pushes code)
- Container destroyed after each task

**PRs require human review/merge** — no unreviewed code lands on main.

See `laptop-setup/2026-05-28-claude-task-queue-design.md` for full implementation design.

## Manual Setup

**All machines:**
- YubiKey SSH key enrollment (verify firmware 5.7+ with `ykman info`)
- Bitwarden setup + migrate from LastPass
- KeePassXC setup for offline vault (recovery codes, vault password)
- `gh auth login`
- Chrome sign-in (bookmarks/extensions)
- Mullvad or ProtonVPN setup

**Work profile:**
- `oc login --web` to Konflux cluster (`kflux-prd-rh02`)
- `podman login registry.redhat.io` (also brew.registry, stage.registry)
- Red Hat entitlements in `/etc/pki/entitlement/*.pem` — needed for RPM lockfile updates. RHEL CSB should have these pre-provisioned; Fedora needs `subscription-manager`.
- `gcloud auth login` + `gcloud auth application-default login`
- `acli` login — 8 config files in `~/.config/acli/`: admin, assets, brie, confluence, global_auth, global, jira, rovodev
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

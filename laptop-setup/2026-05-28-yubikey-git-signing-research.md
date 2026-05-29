---
date: 2026-05-28T15:00:00-04:00
topic: laptop-setup
tags: [yubikey, git-signing, ssh, fido2, claude-code, security]
---

# YubiKey Git Commit Signing Research

Research into git commit signing with YubiKey FIDO2 keys (ed25519-sk),
focused on the interaction between hardware key touch requirements and
Claude Code's automated commit workflow.

## 1. SSH Signing vs GPG Signing

**SSH signing is the clear winner for a YubiKey FIDO2 setup.**

SSH signing (available since git 2.34) is dramatically simpler. Setup takes
about 2 minutes vs the substantial GPG infrastructure required for GPG signing
(GnuPG2, pinentry, key management, subkeys). Since ed25519-sk keys are already
SSH keys, SSH signing uses them directly without any translation layer.

GPG signing advantages:

- Built-in key expiration and revocation
- Broader compatibility across git hosting platforms
- What Linux kernel and git projects themselves use

SSH signing advantages:

- Much simpler setup, especially with existing FIDO2 keys
- No separate GPG infrastructure to maintain
- GitHub, GitLab, and Gitea all support it now
- Same key can serve for both authentication and signing (though using
  separate keys is recommended)

SSH signing disadvantages:

- No built-in expiration or revocation mechanism
- Once verified on GitHub, commits stay verified indefinitely even if the
  key is later deleted

**Recommendation:** Use SSH signing. The simplicity advantage is substantial,
and for a personal workflow the lack of key expiration is manageable.

## 2. Can Claude Code Sign Commits?

**Yes, but there is a critical interaction with YubiKey touch requirements.**

Git commit signing is a local operation. When you run `git commit -S`, git
invokes `ssh-keygen` locally to create the signature. This happens at commit
time, not at push time. The signature is embedded in the commit object itself.

The problem: if the signing key was generated with `-O verify-required` (the
standard security recommendation), every commit requires a physical touch of
the YubiKey. Claude Code makes commits automatically, so it would block
waiting for touch on every commit.

Key insight: **signing happens at commit time, not push time.** You cannot
skip signing locally and have GitHub sign things later (unless you use squash
merge through the GitHub UI, which creates a GitHub-signed commit).

## 3. SSH Signing Touch Requirements

**Standard ed25519-sk keys require touch on every signing operation.**

The `-O verify-required` flag (recommended by Yubico) requires both PIN
entry and physical touch for every cryptographic operation. Even with
ssh-agent running, the touch requirement persists. This would completely
break Claude Code's automated commit workflow.

**The no-touch-required solution:**

Generate a separate key specifically for signing with touch disabled:

```bash
ssh-keygen -t ed25519-sk -O no-touch-required -O resident \
  -C "yubikey-no-touch" -f ~/.ssh/id_ed25519_sk_no_touch
```

This key can sign commits without touch. The YubiKey still must be physically
plugged in (presence is still required at the hardware level), but no finger
tap is needed for each operation.

**Critical distinction: signing vs authentication.**

- Git commit signing uses `ssh-keygen` locally. The no-touch key works here
  because there is no server-side policy enforcing touch.
- GitHub SSH authentication (push/pull) always requires FIDO2 user presence
  (touch). GitHub does not support `no-touch-required` for authentication.
- SSH multiplexing (`ControlMaster auto`, `ControlPersist 600`) means you
  only touch once per 10-minute window for pushes.

**Security tradeoff:** Using `no-touch-required` removes the user-presence
guarantee for signing. The YubiKey must still be plugged in, so physical
access to the machine + YubiKey is still required, but malware on the
machine could theoretically sign commits without the user's knowledge.
This is an acceptable tradeoff for automated agent workflows where the
human is supervising Claude Code's actions.

## 4. GitHub Verified Badge

**Requirements for the green "Verified" badge:**

1. Generate an SSH key (ed25519-sk in our case)
2. Configure git to use SSH signing (see section 7)
3. Upload the **same public key** to GitHub Settings > SSH and GPG keys,
   selecting **"Signing Key"** as the type (not "Authentication Key")
4. The email in `git config user.email` must match a verified email on the
   GitHub account

The same key can be uploaded twice: once as Authentication and once as
Signing. There is no limit on signing keys per account.

**Persistent verification:** Once GitHub verifies a signature at push time,
it stores an immutable verification record. This record persists even if
keys are later rotated, revoked, or contributors leave an organization.
The record includes a timestamp accessible via the REST API.

## 5. Signing vs Verification (Skip Local Signing)

**You can skip local signing and still get verified commits via squash merge.**

How this works in practice:

- Push unsigned commits to feature branches (no branch protection on
  feature branches)
- Create PRs with unsigned commits
- Use "Squash and merge" through the GitHub UI. GitHub signs the resulting
  squash commit with its own GPG key, so the merge commit shows as
  "Verified" even though the original PR commits were unsigned
- Enable "Require signed commits" branch protection only on main/release
  branches

**Important caveat:** This is somewhat of a loophole. GitHub's security
team confirmed that squash merge is allowed because it "condenses the
commits into a single commit signed with GitHub's key." The individual
unsigned commits are discarded. Some projects (like OpenBao) consider
this behavior to make individual commit signing redundant.

**Vigilant mode:** GitHub offers a "Vigilant mode" setting (Settings > SSH
and GPG keys) that flags ALL unsigned commits as "Unverified." Without
vigilant mode, unsigned commits simply show no badge at all. With it
enabled, unsigned commits get an explicit warning. Only enable this if
you commit to signing everything.

## 6. Best Practice for AI Agent Commits

**The field is still converging on a standard. Current approaches:**

**Co-Authored-By trailer (most common, but flawed):**

```text
Co-Authored-By: Claude <noreply@anthropic.com>
```

Problems: semantic mismatch (AI is a tool, not a co-author), the
`noreply@anthropic.com` email has caused GitHub to attribute commits to
random users, the model name in the trailer is sometimes wrong, and the
format keeps changing across releases.

**Better alternatives emerging:**

- `Assisted-by: Claude Code` trailer (aligns with git's existing `-by`
  convention like `Reported-by`, `Reviewed-by`)
- `--author` flag to separate git author (AI) from committer (human)
- RAI Footers: pair `Assisted-by:` with `Signed-off-by:` to show human
  review of AI output
- Separate trailers for tool and model:
  `Coding-Agent: Claude Code` / `Model: claude-opus-4-6`

**For signing specifically:** The human who supervises the AI agent should
sign the commits. The commit signature attests that a trusted party
authorized the change, not that they typed every character. This is
analogous to a senior engineer signing off on a junior engineer's work.

**Practical recommendation:** Use `Signed-off-by` (the `-s` flag, already
configured in this repo's CLAUDE.md) for human accountability, and let
Claude Code add its `Co-Authored-By` or `Assisted-by` trailer for
transparency. Sign commits with the no-touch YubiKey key so every commit
has cryptographic proof of the human's involvement.

## 7. Git Config for SSH Signing

**Complete gitconfig for SSH signing with YubiKey no-touch key:**

```ini
[user]
    name = Daniel Farrell
    email = dfarrell@redhat.com
    signingkey = ~/.ssh/id_ed25519_sk_no_touch.pub

[gpg]
    format = ssh

[gpg "ssh"]
    allowedSignersFile = ~/.config/git/allowed_signers

[commit]
    gpgsign = true

[tag]
    gpgsign = true
```

**Set up the allowed signers file for local verification:**

```bash
mkdir -p ~/.config/git
echo "$(git config --get user.email) $(cat ~/.ssh/id_ed25519_sk_no_touch.pub)" \
  >> ~/.config/git/allowed_signers
```

**Verify it works:**

```bash
git commit --allow-empty -m "Test signed commit"
git log --show-signature -1
```

## 8. Two-Key Setup for Claude Code

**The recommended architecture (from azevedo-home-lab/claude-code-workflows):**

| Operation | Key | Touch Required? |
| --- | --- | --- |
| git commit (signing) | no-touch key | No |
| git push (first in session) | touch key | Yes (one touch) |
| git push (subsequent) | reused connection | No (SSH multiplexing) |
| YubiKey unplugged | blocked | All operations fail |

**SSH config for multiplexing:**

```text
Host github.com
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

**Key generation commands:**

```bash
# Touch-required key for authentication (if not already created)
ssh-keygen -t ed25519-sk -O resident -O verify-required \
  -C "yubikey-touch" -f ~/.ssh/id_ed25519_sk

# No-touch key for signing
ssh-keygen -t ed25519-sk -O no-touch-required -O resident \
  -C "yubikey-no-touch-signing" -f ~/.ssh/id_ed25519_sk_no_touch
```

**Register both on GitHub:**

```bash
# Signing key
gh ssh-key add ~/.ssh/id_ed25519_sk_no_touch.pub \
  --title "yubikey-no-touch (signing)" --type signing

# Authentication key (if not already registered)
gh ssh-key add ~/.ssh/id_ed25519_sk.pub \
  --title "yubikey-touch (auth)" --type authentication
```

**Create the SSH sockets directory:**

```bash
mkdir -p ~/.ssh/sockets
```

## 9. Linux-Specific Notes (Fedora)

**Required packages:**

```bash
sudo dnf install -y openssh libfido2
```

**Known issue:** GNOME Keyring's SSH agent does not understand FIDO2/ed25519-sk
keys. It intercepts the request and refuses the operation with "agent refused
operation." Fix: ensure the YubiKey SSH operations go through OpenSSH's
ssh-agent, not GNOME Keyring's agent.

**Resident key import caveat:** The `no-touch-required` flag is NOT restored
when importing keys from a YubiKey via `ssh-keygen -K`. The flag only exists
in the original private key handle file. Back up `~/.ssh/id_ed25519_sk_no_touch`
carefully.

## 10. Decision Summary

For this setup (YubiKey 5C NFC, firmware 5.7+, Fedora, Claude Code):

1. **Use SSH signing** (not GPG) -- simpler, works directly with ed25519-sk
2. **Generate a no-touch key** for commit signing -- enables Claude Code to
   sign commits without blocking on touch
3. **Keep the touch key** for push authentication -- maintains security for
   the destructive operation of pushing code
4. **Use SSH multiplexing** -- reduces push touches to once per 10 minutes
5. **Upload the no-touch key** to GitHub as a Signing Key
6. **Enable `commit.gpgsign = true`** globally
7. **Keep `--signoff`** for human accountability alongside cryptographic signing
8. **Do not enable vigilant mode** until confident every commit path is signing
9. **Consider the squash-merge fallback** -- even if some commits slip through
   unsigned, squash merge through GitHub UI produces verified commits

## Sources

- [Yubico: Securing git with SSH and FIDO2](https://developers.yubico.com/SSH/Securing_git_with_SSH_and_FIDO2.html)
- [Yubico: Securing SSH with FIDO2](https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html)
- [GitHub Docs: About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
- [GitHub Docs: Vigilant mode](https://docs.github.com/en/authentication/managing-commit-signature-verification/displaying-verification-statuses-for-all-of-your-commits)
- [GitHub Docs: Telling Git about your signing key](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key)
- [Comparing GitHub Commit Signing Options (Ken Muse)](https://www.kenmuse.com/blog/comparing-github-commit-signing-options/)
- [Claude Code YubiKey workflows (azevedo-home-lab)](https://github.com/azevedo-home-lab/claude-code-workflows/tree/main/tools/yubikey-setup)
- [Claude Code marketplace YubiKey issue #16274](https://github.com/anthropics/claude-code/issues/16274)
- [Why Co-Authored-By falls short (Fabio Rehm)](https://fabiorehm.com/blog/2026/03/02/our-coding-agent-commits-deserve-better-than-co-authored-by/)
- [RAI Footers for AI attribution](https://dev.to/anchildress1/signing-your-name-on-ai-assisted-commits-with-rai-footers-2b0o)
- [Attribute Git Commits to AI Agents](https://elite-ai-assisted-coding.dev/p/attribute-git-commits-to-ai-agents)
- [Using YubiKey 5C on Linux (thenets.org)](https://thenets.org/posts/using-a-yubikey-5c-for-ssh-and-git-authentication-on-linux/)
- [GitHub no-touch-required feature request](https://github.com/orgs/community/discussions/10593)
- [Unsigned commits via squash merge (Mergify)](https://articles.mergify.com/un-signed-commits-how-we-found-a-non-security-bug-in-github/)
- [OpenBao: Discontinue enforcing signed commits](https://github.com/openbao/openbao/issues/399)

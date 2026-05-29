---
date: 2026-05-29T08:30:00-04:00
topic: laptop-setup
tags: [security, supply-chain, ansible, git-signing, ansible-galaxy, github-actions, integrity-verification]
---

# Supply Chain Protection for Ansible Workstation Provisioning

Research into protecting a `laptop-setup` Ansible repository that runs with
`become: true` (sudo) on fresh machines. If an attacker compromises this repo,
they get full root access to every machine provisioned with it.

## Threat Model Summary

The repo lives on GitHub. The user runs `make all` which calls
`ansible-playbook site.yml` with privilege escalation. Attack vectors include:
GitHub account takeover, force push, malicious PR merge, dependency confusion
in Ansible Galaxy collections, compromised CI workflows, and man-in-the-middle
during collection download.

---

## 1. Commit Signing Enforcement

**Can GitHub require signed commits?** Yes. Branch protection rules include a
"Require signed commits" option that blocks merging or pushing any unsigned
commit. GitHub supports GPG, SSH, and S/MIME signing.

**Does this protect against account takeover?** Partially. If an attacker
compromises your GitHub credentials (password, session token), they can push
commits through the web UI or API. GitHub itself signs web-based commits with
GitHub's own GPG key, so those show as "Verified." However, if you require
commits to be signed with *your specific GPG key* and the attacker does not
have your private key, they cannot produce valid signatures from the CLI.

**Limitations:**

- GitHub's "Require signed commits" only checks that *a* valid signature
  exists, not that it came from a *specific* key. An attacker with their own
  GPG key added to their own GitHub account could still produce "Verified"
  commits if they gain push access.
- Squash-and-merge commits are signed by GitHub itself, making individual
  commit signatures moot.
- Rebase-and-merge creates unsigned commits because GitHub cannot sign on your
  behalf.
- The real protection comes from combining signed commits with a YubiKey or
  hardware-backed GPG key that the attacker cannot extract remotely.

**Recommendation:** Enable "Require signed commits" on the main branch. Use a
hardware-backed GPG key (YubiKey). This means even if someone steals your
GitHub session token, they cannot produce commits signed with your key.

Sources:

- [GitHub: About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
- [Anyone can commit as you on GitHub](https://dev.to/nickytonline/anyone-can-commit-code-as-you-on-github-heres-how-to-stop-them-2in7)

---

## 2. Dependency Pinning in requirements.yml

**Are pinned versions verified by hash?** No. Ansible Galaxy `requirements.yml`
pins by version number only (e.g., `version: ">=9.0.0,<10.0.0"`). There is no
hash pinning equivalent to pip's `--hash` or npm's `integrity` field.

**Can an attacker replace a pinned version?** Yes. Published versions on
galaxy.ansible.com can be overwritten by the publisher because there is no
immutability enforcement on the registry side. This is a known, documented
weakness. A lockfile feature was first requested in 2016 and remains unimplemented
as of 2026 (the now-archived Mazer tool briefly had `install --lockfile` before
being abandoned in 2020).

**What verification exists?**

- `ansible-galaxy collection verify` can check checksums against the server,
  but it is opt-in and off by default.
- GPG signature verification exists via `--keyring` but is also opt-in and
  off by default.
- Neither checksum nor signature verification runs during normal
  `ansible-galaxy collection install`.

**Recommendation:** After installing collections, run
`ansible-galaxy collection verify` to detect changes. Consider downloading
collection tarballs, verifying their checksums manually, and installing from
the local tarball rather than from Galaxy directly. Store the verified tarballs
in the repo or a trusted artifact store.

Sources:

- [Ansible docs: Verifying collections](https://docs.ansible.com/ansible/latest/collections_guide/collections_verifying.html)
- [Andrew Nesbitt: If It Quacks Like a Package Manager](https://nesbitt.io/2026/03/08/if-it-quacks-like-a-package-manager.html)
- [Galaxy collection overwrite bug](https://github.com/ansible/galaxy_collection/issues/139)

---

## 3. Ansible Galaxy Supply Chain: Who Maintains the Collections?

### community.general

- Maintained by volunteer community members under the `ansible-collections`
  GitHub organization.
- Maintainer list is in `commit-rights.md` in the repo.
- Part of the official Ansible package, so Red Hat has some oversight, but
  day-to-day maintenance is community-driven.
- No known compromise as of May 2026.

### containers.podman

- Maintained under the `containers` GitHub organization (same org as Podman,
  Buildah, Skopeo).
- Primary author is Sagi Shnaidman (Red Hat).
- Currently in maintenance mode, accepting mostly bugfixes and security patches.
- No known compromise as of May 2026.

### Known Ansible Galaxy Supply Chain Weaknesses

- Researchers demonstrated typosquatting on Galaxy (e.g., `hashic0rp/aws` with
  a zero instead of an "o").
- A live supply chain attack demonstration at NDC Oslo 2025 showed typosquatting
  working in practice.
- Academic research (Konala et al., DBSec 2025) analyzed 482 Galaxy repositories
  and found vulnerabilities in 45 dependency chains including deprecated
  dependencies, hardcoded credentials, and improper file permissions.
- Roles execute with the full privileges of the Ansible process, and `become`
  directives escalate further.
- There are open issues going back years about the inability to exclude or
  override transitive role dependencies.

**No known incident of a malicious Ansible Galaxy collection being used in an
active supply chain attack** has been publicly reported as of May 2026. But the
structural weaknesses make it feasible.

Sources:

- [community.general on GitHub](https://github.com/ansible-collections/community.general)
- [containers.podman on GitHub](https://github.com/containers/ansible-podman-collections)
- [Konala et al.: Metadata Assisted Supply-Chain Attack Detection for Ansible (DBSec 2025)](https://link.springer.com/chapter/10.1007/978-3-031-96590-6_18)
- [Andrew Nesbitt: If It Quacks Like a Package Manager](https://nesbitt.io/2026/03/08/if-it-quacks-like-a-package-manager.html)

---

## 4. GitHub Actions on the Repo

**Should CI be disabled?** It depends on what the CI does. The risks:

- A compromised GitHub Actions workflow can modify files in the repo, exfiltrate
  secrets, or inject malicious code into the playbook.
- Unpinned action references (using `@v4` instead of a commit SHA) are mutable
  and can be replaced by an attacker who compromises the action's repo.
- The `tj-actions/changed-files` compromise (2025) affected over 20,000
  repositories. A new maintainer published a version with obfuscated malicious
  code.
- The Trivy GitHub Action compromise (2026) saw attackers force-push 75 out of
  76 version tags.
- Self-hosted runners are especially dangerous and should never be used for
  public repos.

**For a personal workstation provisioning repo:**

- If CI only runs linting (yamllint, ansible-lint), the risk is low but not
  zero.
- If CI has write access to the repo (can push commits, create releases), it
  is a significant risk.
- Any Actions workflow that runs on `pull_request_target` from forks is
  dangerous because it executes with the privileges of the base repo.

**Recommendation:** If you keep CI, pin all actions to full commit SHAs (not
tags). Remove any workflow that has write permissions. Consider whether the CI
provides enough value to justify the attack surface. For a personal provisioning
repo, linting locally before committing may be sufficient.

Sources:

- [Wiz: Hardening GitHub Actions](https://www.wiz.io/blog/github-actions-security-guide)
- [InfoQ: Compromised GitHub Action highlights risks](https://www.infoq.com/news/2025/04/compromised-github-action/)
- [Security Boulevard: GitHub Actions Supply Chain Attack](https://securityboulevard.com/2026/04/github-actions-supply-chain-attack-trivy-breach-workflow/)
- [Sysdig: Self-hosted runners as backdoors](https://www.sysdig.com/blog/how-threat-actors-are-using-self-hosted-github-actions-runners-as-backdoors)

---

## 5. Fork Protection

**If someone forks and submits a PR, what protections exist?**

- Branch protection can require reviews before merging, but for a single-user
  repo you cannot meaningfully review your own code (GitHub blocks self-approval
  of PRs).
- A security researcher at Cider Security demonstrated that GitHub Actions can
  be used to bypass review requirements: an attacker pushes malicious code plus
  a workflow that auto-approves the PR via the GitHub API.
- "Pull request hijacking" allows any authorized reviewer to modify an existing
  PR, then approve and merge it, because GitHub does not track that a different
  user modified the PR.

**Mitigations:**

- Disable "Allow GitHub Actions to create and approve pull requests" in repo
  settings.
- Enable "Dismiss stale pull request approvals when new commits are pushed."
- Enable "Require review of the most recent reviewable push."
- For a single-user repo, the strongest protection is simply not accepting PRs
  and disabling fork PRs entirely.

Sources:

- [Cider Security: Bypassing required reviews using GitHub Actions](https://medium.com/cider-sec/bypassing-required-reviews-using-github-actions-6e1b29135cc7)
- [Legit Security: Attackers can bypass GitHub required reviewers](https://www.legitsecurity.com/blog/bypassing-github-required-reviewers-to-submit-malicious-code)
- [GitHub: Prevent self-reviews for secure deployments](https://github.blog/changelog/2023-10-16-actions-prevent-self-reviews-for-secure-deployments-across-actions-environments/)

---

## 6. Playbook Integrity Verification

**Can the playbook be verified before running on a fresh machine?** Yes.
Red Hat developed `ansible-sign`, a CLI tool for GPG-based project signing and
verification.

**How it works:**

1. Create a `MANIFEST.in` file listing which files to track.
2. Run `ansible-sign project gpg-sign .` to generate checksums and a detached
   GPG signature in a `.ansible-sign/` directory.
3. Before running on a fresh machine, run
   `ansible-sign project gpg-verify .` to verify the signature and that no
   files have been modified.
4. If verification fails, the playbook should not run.

**What gets created:**

```text
.ansible-sign/
  sha256sum.txt      # Checksum manifest of every tracked file
  sha256sum.txt.sig  # Detached GPG signature of the manifest
```

**Integration with Ansible Automation Platform (AAP):** AAP/AWX can be
configured to verify project signatures automatically. If verification fails,
the project update fails and no jobs can launch.

**For the laptop-setup scenario:**

- Sign the project after every commit using your GPG key.
- On the fresh machine, import your public key and run `ansible-sign project
  gpg-verify .` before running the playbook.
- This can be wrapped in the Makefile: verify first, then run.

There is also an IBM-developed collection (`IBM/playbook-integrity-collection`)
that provides signing and verification as Ansible playbooks themselves.

Sources:

- [Red Hat: Project signing and verification](https://www.redhat.com/en/blog/project-signing-and-verification)
- [Ansible Sign CLI documentation](https://docs.ansible.com/projects/sign/en/latest/rundown.html)
- [Red Hat AAP 2.5: Project Signing and Verification](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/assembly-controller-project-signing)
- [IBM Playbook Integrity Collection](https://github.com/IBM/playbook-integrity-collection)

---

## 7. Two-Person Rule for Single-User Repos

**The fundamental problem:** GitHub does not allow you to approve your own PRs.
For a single-user repo, requiring one approval means you are permanently blocked.

**Options:**

1. **Remove the review requirement.** Accept the risk and use other controls
   (signed commits, playbook integrity verification) instead.
2. **Add a trusted second GitHub account as a reviewer.** This could be a
   partner, colleague, or a dedicated "security review" account you control
   on a separate device. This is security theater if you control both accounts
   on the same machine.
3. **Use GitHub Environments with approval gates.** If you use GitHub Actions
   for deployment, you can require manual approval before a deployment job runs.
   But this does not apply to "clone and run locally" workflows.
4. **Time delay.** Commit changes, wait 24-48 hours, then review your own diff
   before running on a fresh machine. Not a technical control, but catches
   mistakes and gives time to detect unauthorized changes.
5. **Separate signing machine.** Only sign the project from a dedicated,
   hardened machine. This means even if your GitHub account is compromised,
   unsigned code will not pass verification.

**Recommendation:** For a personal repo, the most practical approach is
combining signed commits (with a hardware key) and `ansible-sign` verification.
The signing key acts as the "second person" -- the attacker needs both your
GitHub access and your physical hardware key.

---

## 8. Git Hook Protection

**Can a compromised `.git/hooks/` attack during clone?** Not through a normal
`git clone`. Git intentionally does not copy hooks from the remote repository's
`.git/hooks/` directory. This is a deliberate security design decision.

**However, there are attack vectors:**

- **CVE-2024-32002 (June 2024):** A crafted repository with submodules could
  trick Git into writing files into `.git/` instead of the submodule's worktree,
  allowing a hook to execute during clone. Affected Git versions before 2.45.1.
  Requires symlink support on case-insensitive filesystems.
- **Lazarus Group "Contagious Interview" campaign:** North Korean attackers
  embed malware in `.git/hooks/pre-commit` scripts, then distribute the entire
  repository directory (including `.git/`) via Google Drive or ZIP files rather
  than via `git clone`. This bypasses Git's protection because the hooks are
  already in place when the user opens the directory.
- **`core.hooksPath` configuration:** If a repo's `.gitconfig` or the user's
  global config points `core.hooksPath` to a tracked directory (e.g.,
  `.githooks/`), then hooks in that directory will run. However, this requires
  the user to explicitly configure it.

**For the laptop-setup repo:** The normal `git clone` workflow is safe from
hook injection. The risk arises if you distribute the repo via ZIP/tarball that
includes `.git/`, or if you configure `core.hooksPath` to point at a tracked
directory. Neither is likely for this use case.

**Recommendation:** Always use `git clone` (not ZIP downloads) to get the repo.
Keep Git updated (2.45.1+ for CVE-2024-32002 fix). Avoid setting
`core.hooksPath` to a directory inside the repo.

Sources:

- [Lazarus Group git hooks malware](https://www.msbiro.net/posts/lazarus-group-git-hooks-malware-developers/)
- [CVE-2024-32002: RCE via git clone](https://amalmurali.me/posts/git-rce/)
- [Git hooks security and distribution](https://learning-ocean.com/tutorials/git/git-hooks-security-and-distribution/)

---

## 9. ansible-galaxy Integrity Verification

**Does `ansible-galaxy install` verify checksums or signatures?** Not by
default. Verification is entirely opt-in.

**Available verification mechanisms:**

| Mechanism | Command | Default |
| --------- | ------- | ------- |
| Checksum verification | `ansible-galaxy collection verify` | Off (opt-in) |
| GPG signature verification | `--keyring` flag during install | Off (opt-in) |
| Required signature count | `--required-valid-signature-count` | 1 (if verification enabled) |
| Sigstore integration | Proposed, not yet implemented | N/A |

**Key problems:**

- Normal `ansible-galaxy collection install` performs no checksum or signature
  verification.
- Published versions on Galaxy can be overwritten by the publisher (no
  immutability).
- Roles sourced from git repos use mutable tags (same problem as GitHub Actions
  tag references).
- No lockfile support exists despite being requested since 2016.
- Transitive dependencies install automatically with the same privilege level.

**Recommendation:** After installing collections, immediately run:

```bash
ansible-galaxy collection verify community.general
ansible-galaxy collection verify containers.podman
```

For stronger guarantees, download collection tarballs manually, compute SHA-256
checksums, record them in a checked-in file, and verify before each install.
Consider vendoring collections directly in the repo.

Sources:

- [Ansible docs: Verifying collections](https://docs.ansible.com/ansible/latest/collections_guide/collections_verifying.html)
- [Sigstore verification proposal](https://github.com/ansible/galaxy/issues/3126)
- [Andrew Nesbitt: If It Quacks Like a Package Manager](https://nesbitt.io/2026/03/08/if-it-quacks-like-a-package-manager.html)

---

## 10. Best Practices from Other IaC Projects

### Terraform (HashiCorp)

- **Provider signing:** Terraform automatically verifies GPG signatures of
  providers during `terraform init`.
- **Lock files:** `.terraform.lock.hcl` records provider versions and checksums.
  Committed to version control.
- **Module verification:** Modules are NOT cryptographically verified. Pin
  module versions and use private registries.
- **The Codecov incident (April 2021):** The Codecov Bash Uploader was
  compromised for two months, exfiltrating CI environment variables.
  HashiCorp's GPG private key for signing releases was exposed. They rotated
  the key and re-signed all releases. This is the canonical example of how a
  CI tool compromise can undermine IaC signing infrastructure.

### Puppet

- Puppet modules from the Forge are not cryptographically signed.
- Trust relies on module author reputation and Puppet Forge moderation.
- Enterprise deployments typically use internal module repositories with
  access controls.

### General IaC Best Practices

| Practice | Terraform | Ansible | Puppet |
| -------- | --------- | ------- | ------ |
| Provider/collection signing | Yes (GPG, automatic) | Opt-in (GPG) | No |
| Dependency lock file | Yes (.terraform.lock.hcl) | No | No |
| Version pinning | Yes | Yes (version only, no hash) | Yes |
| Immutable published versions | Yes (registry) | No (can be overwritten) | Partial |
| Policy-as-code enforcement | Sentinel, OPA | None built-in | None built-in |

Sources:

- [HashiCorp: Codecov GPG key exposure advisory](https://discuss.hashicorp.com/t/hcsec-2021-12-codecov-security-event-and-hashicorp-gpg-key-exposure/23512)
- [BleepingComputer: HashiCorp Codecov attack](https://www.bleepingcomputer.com/news/security/hashicorp-is-the-latest-victim-of-codecov-supply-chain-attack/)
- [Sysdig: Terraform security best practices](https://www.sysdig.com/blog/terraform-security-best-practices)
- [Wiz: Terraform security best practices](https://www.wiz.io/academy/terraform-security-best-practices)
- [GitLab: Terraform in the software supply chain](https://about.gitlab.com/blog/terraform-as-part-of-software-supply-chain-part1-modules-and-providers/)

---

## Real-World IaC/Developer Tool Compromise Incidents

### Codecov Bash Uploader (January-April 2021)

- Attackers modified the Codecov Bash Uploader to exfiltrate CI environment
  variables (tokens, keys, credentials) from customer CI pipelines.
- Ran undetected for approximately two months.
- HashiCorp's GPG signing key was exposed. Hundreds of customer networks
  were reportedly breached.
- U.S. federal investigators were involved.

### tj-actions/changed-files (2025)

- A maintainer published a version with obfuscated malicious code.
- Affected over 20,000 repositories that referenced the action.
- Demonstrated that mutable tag references (`@v4`) in GitHub Actions are a
  single point of failure.

### Trivy GitHub Action (February-April 2026)

- Attackers force-pushed 75 out of 76 version tags in the Trivy GitHub Action.
- Over 10,000 repositories referenced the compromised action.
- Led to cascading compromise of Checkmarx KICS (IaC scanner), which scans
  Terraform, CloudFormation, and Kubernetes configs that may contain credentials.

### Megalodon Campaign (May 2026)

- Compromised 5,561 public GitHub repositories by injecting malicious CI/CD
  workflows.
- Worm-like propagation: compromised tokens were used to target additional
  repos in a cycle.
- Once a repo owner merged a malicious commit, the injected workflow
  exfiltrated secrets and enabled lateral movement.

### GitHub Internal Repos Breach via Nx Console Extension (May 2026)

- Compromised VS Code extension was live for 11-18 minutes.
- Approximately 3,800 internal GitHub repositories were exfiltrated.
- Internal repos contained infrastructure configurations, deployment scripts,
  staging credentials, and internal API schemas.

### Checkmarx KICS IaC Scanner (March-April 2026)

- TeamPCP used stolen GitHub PATs to force-push malicious commits to all 35
  version tags of `checkmarx/kics-github-action`.
- The malware generated uncensored scan reports, encrypted them, and sent them
  to an external endpoint.
- Organizations that scanned Terraform/CloudFormation/K8s configs with the
  compromised image should treat all exposed secrets as compromised.

### Microsoft Durable Task Python SDK (May 2026)

- Three malicious versions published to PyPI within a 35-minute window.
- Payload stole credentials from AWS, Azure, GCP, Kubernetes, and 90+
  developer tool configurations, then spread laterally through cloud
  infrastructure.

Sources:

- [Filippo Valsorda: Retrospective survey of open source compromises](https://words.filippo.io/compromise-survey/)
- [Palo Alto Unit 42: TeamPCP supply chain attacks](https://unit42.paloaltonetworks.com/teampcp-supply-chain-attacks/)
- [VentureBeat: GitHub 3,800 repos stolen](https://venturebeat.com/security/github-confirms-3800-repos-stolen-poisoned-vs-code-extension-supply-chain-worm-microsoft-python-sdk)
- [Rescana: Megalodon campaign](https://www.rescana.com/post/megalodon-supply-chain-attack-teampcp-compromises-5-561-github-repositories-via-malicious-ci-cd-workflows)

---

## Recommended Defense Layers for laptop-setup

Listed in priority order (highest impact first):

### Layer 1: Signed Commits with Hardware Key

- Use a YubiKey-backed GPG key for all commits.
- Enable "Require signed commits" on the main branch.
- Attacker needs physical hardware key even if they compromise GitHub account.

### Layer 2: Playbook Integrity Verification

- Use `ansible-sign` to sign the project after every change.
- Before running on a fresh machine, verify the signature.
- Wrap in the Makefile: `make verify && make all`.

### Layer 3: Vendor or Pin Dependencies by Hash

- Download collection tarballs and store checksums in a tracked file.
- Or vendor collections directly in the repo to eliminate Galaxy as an
  attack surface entirely.
- Verify checksums before installing.

### Layer 4: Minimize GitHub Attack Surface

- Disable GitHub Actions if CI is not essential.
- If keeping CI, pin all actions to commit SHAs.
- Disable fork PRs or disable Actions PR approval.
- Enable branch protection with "Do not allow bypassing the above settings."

### Layer 5: Verify Before Running

- On a fresh machine, before running the playbook:
  1. Verify GPG signature on the latest commit.
  2. Run `ansible-sign project gpg-verify .`
  3. Verify collection checksums.
  4. Only then run `make all`.

### Layer 6: Limit Blast Radius

- Separate privileged and unprivileged tasks into different playbooks.
- Run unprivileged tasks first (dotfiles, user-level config) without `become`.
- Only use `become: true` for tasks that genuinely require root.
- Review the privileged playbook separately before running.

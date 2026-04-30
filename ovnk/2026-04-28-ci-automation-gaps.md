---
date: 2026-04-28T18:00:00Z
topic: ovnk
tags: [ci, automation, gaps, future-work, submariner-patterns, first-prs, security, verified, dependency-management, linting]
---

# OVNK CI/Automation Gaps - All Contribution Opportunities

Every item verified by reading source files + running actual tools against the
codebase. Violation counts are real. Items that failed verification are at the end.

---

## First PR — ovn-kubernetes upstream (VERIFIED, ready to submit)

- [ ] **Update Alpine 3.10 → 3.21 in retest-action** — `.github/actions/retest-action/Dockerfile:4`. Only uses curl+jq, pure POSIX shell. Alpine 3.10 EOL since May 2021. Risk: zero.
- [ ] **Add top-level permissions to performance-test.yml** — Add `permissions: contents: read` after line 9. `performance-job` already has job-level overrides. Risk: zero.
- [ ] **Add top-level permissions to pr-labeler.yml** — Add `permissions: contents: read` + `pull-requests: write` after line 3. Identical to existing job-level block. Risk: zero.
- [ ] **Add paths-ignore to performance-test.yml** — Add `paths-ignore: ['**/*.md', 'mkdocs.yml']` to `pull_request` trigger only. Saves 32-CPU Oracle VM runners on docs-only PRs.

## Second PR — ovn-kubernetes (VERIFIED, needs care)

- [ ] **Add permissions to commands.yml** — Needs `actions: write`, `issues: write`, `pull-requests: read` after line 4. `actions: write` critical for `/retest`. Risk: moderate.

## Third PR — SHA pinning + dependabot (VERIFIED, pairs together)

- [ ] **Pin all 17 external GHA actions to SHAs** — 76 references across 8 files. Zero existing pins.
- [ ] **Add dependabot.yml** — Cover github-actions + all 3 go.mod dirs (go-controller/, test/e2e/, test/conformance/ — confirmed). Use grouped updates. SECURITY.md claims dependabot but no config exists.

## Other repos — small verified fixes

- [ ] **frr-k8s: Pin cosign-installer@main to SHA** — `publish.yaml:52`. Release pipeline with quay.io creds + OIDC tokens.
- [ ] **cluster-network-operator: Fix godoc.org → pkg.go.dev link** — `README.md:5`. 301 redirect, anchor lost.

---

## Security Linting (VERIFIED — actual violation counts)

### gosec — 50 findings, easy PR

VERIFIED by running golangci-lint with gosec enabled. 900 Go files, 208k lines.

| Rule | Count | Description | Action |
|------|-------|-------------|--------|
| G115 | 22 | Integer overflow conversions | Suppress — bounded network values |
| G402 | 1 | **TLS MinVersion too low (TLS 1.0)** | **Fix — genuine security issue** in `cmd/ovnkube-identity/ovnkubeidentity.go:376` |
| G118 | 3 | Context cancellation not called | Fix — real issue in pkg/cni/ |
| G112 | 3 | No ReadHeaderTimeout (slowloris) | Fix — add timeout to 3 http.Server instances |
| G304 | 3 | File inclusion via variable | Suppress — known safe paths |
| G104 | 2 | Unhandled errors | Fix — in test helpers |
| Others | 16 | Misc (permissions, weak random, etc.) | Mix of fix and suppress |

- [ ] **Add gosec to golangci.yml** — Single PR. Fix G402 TLS issue + G112 slowloris + G118 context. Suppress G115 bulk. ~25 fixes + ~20 suppressions.

### govulncheck — 0 vulnerabilities, trivial

VERIFIED by running `govulncheck ./...`. Zero current violations.

- [ ] **Add govulncheck to CI** — Add as workflow step. Catches future dep vulns. No fixes needed today.

### CodeQL — optional, modest incremental value

- [ ] **Add CodeQL SAST** — Free for public repos. 208k lines scans in 5-15 min. Overlap with gosec is significant, so incremental value is modest. multus-cni has it (v2, should upgrade to v3).

### Container scanning — practical

- [ ] **Container vulnerability scanning (Grype/Trivy)** — docker.yml pushes real images to ghcr.io on every merge. Post-build scan or separate scheduled workflow.

---

## Shell Linting (VERIFIED — actual violation counts)

### shellcheck — 18 errors, 908 total, practical with phased approach

VERIFIED by running shellcheck on all 31 scripts.

| Severity | Count | Notes |
|----------|-------|-------|
| Error | 18 | Missing shebangs (4), unquoted arrays (13), multi-param shebang (1) |
| Warning | 118 | Various |
| Note | 772 | SC2086 accounts for 659 (73%) |

Worst offenders: `ovnkube.sh` (414), `ovndb-raft-functions.sh` (170), `kind-common.sh` (75).

- [ ] **Add shellcheck -S error to CI** — Fix 18 errors in 9 scripts (missing shebangs, unquoted arrays). 22 of 31 scripts already pass at error level. Single PR. Then progressively tighten.

---

## Go Linter Additions (not yet run — estimates)

Low-noise candidates from Submariner's ~70 linters. Each should be tested before proposing.

- [ ] **misspell** — Typos in comments/strings. Typically low noise, easy fixes.
- [ ] **bodyclose** — Unclosed HTTP response bodies. Few false positives.
- [ ] **contextcheck** — context.Context misuse. Catches real bugs.
- [ ] **errchkjson** — Unchecked json.Marshal errors.
- [ ] **copyloopvar** — Loop variable capture. CNO already enables this.
- [ ] **errorlint** — Error wrapping. May be noisy initially.
- [ ] **wrapcheck** — External errors wrapped. Potentially noisy.
- [ ] **gocritic** — Broad suggestions. Very configurable.

---

## Non-Go Linting (VERIFIED feasibility)

- [ ] **markdown-link-check** — VERIFIED: 126 md files, 400 links. Non-invasive, catches dead links.
- [ ] **hadolint** — VERIFIED: 5 Dockerfiles. No repo in ecosystem does this. Feasible scope.
- [ ] **yamllint** — Not yet assessed. Many YAML files, likely many existing violations.
- [ ] **markdownlint** — Not yet assessed. Workspace CI has it for top-level only.

---

## Dependency Management

### Current state (6 of 36 repos have dependabot)

| Repo | Ecosystems | Schedule |
|------|-----------|----------|
| libovsdb | gomod + github-actions | weekly, grouped |
| kubernetes-mcp-server | gomod + github-actions | daily, grouped |
| frr-k8s | gomod + github-actions | gomod daily, GHA weekly |
| kubernetes-enhancements | gomod + github-actions | daily, limit 10 |
| network-policy-api | github-actions + docker | weekly |
| ai-helpers | github-actions only | weekly |
| **ovn-kubernetes** | **NONE** | **CVE alerts only (org-level)** |

### What's automated vs manual

| Dependency | Method | Who |
|------------|--------|-----|
| Go transitive deps (CVEs) | Dependabot (org-level) | dependabot[bot], trozet merges |
| K8s rebases | Manual (15+ go get × 3 dirs) | Meina-rh, Arti Sood, Jaime |
| libovsdb bumps | Manual | Dave Tucker, Tim Rozet |
| OVN/OVS version | Manual Dockerfile edit | Surya, Patryk Diak |
| GHA action versions | Manual | Nadia Pinaeva |
| Upstream→downstream sync | Semi-automated (hourly Prow periodic) | Jamo Luhrsen, Jaime |

### Proposed dependabot.yml

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      github-actions:
        patterns: ["*"]
  - package-ecosystem: "gomod"
    directory: "/go-controller"
    schedule:
      interval: "weekly"
    groups:
      go-dependencies:
        patterns: ["*"]
  - package-ecosystem: "gomod"
    directory: "/test/e2e"
    schedule:
      interval: "weekly"
    groups:
      go-dependencies:
        patterns: ["*"]
  - package-ecosystem: "gomod"
    directory: "/test/conformance"
    schedule:
      interval: "weekly"
    groups:
      go-dependencies:
        patterns: ["*"]
```

---

## Linting Ecosystem Comparison

| Repo | golangci linters | gosec | govulncheck | shellcheck | yamllint | commitlint |
|------|-----------------|-------|-------------|------------|----------|------------|
| **Submariner** | ~70 | yes | via Grype | yes | yes | gitlint |
| **frr-k8s** | 14 | yes | yes | no | no | commitlint |
| **CNO** | 12 | no | no | no | no | no |
| **ovn-kubernetes** | 11 | no | no | no | no | no |
| **libovsdb** | 9 | no | no | no | no | no |
| **multus-cni** | revive only | no | no | no | no | no |

---

## Larger Automation Projects (Submariner pattern reuse)

- [ ] **AGENTS.md for upstream ovnk** — PR #5597 confirmed stalled (lifecycle/stale label, no activity since 2026-03-06). trozet wants vendor-neutral name.
- [ ] **CVE/dependency scanning skill** — Adapt Submariner `cve-fix` skill. ~80% reuse.
- [ ] **Automated release workflow** — Manual releases exist (v1.0.0 through v1.2.0 via tag + GitHub UI), but no automated workflow. Could add changelog generation, GitHub Release creation on tag push.
- [ ] **CI failure triage skill** — Adapt `konflux-ci-fix` for Prow. Highest single-impact.
- [ ] **Test flake management** — Build skill using sippy MCP.
- [ ] **Container image signing** — Add Trivy scan + cosign signing to docker.yml.
- [ ] **Multi-K8s version testing** — VERIFIED: K8S_VERSION hardcoded to v1.35.0 in test.yml:23, performance-test.yml:12, install-kind.sh:51. Could parameterize as matrix.

---

## Jamo Luhrsen — Context for Collaboration

Jamo (jluhrsen) is the primary downstream merge shepherd and OTE test architect.
Understanding his work helps us contribute in ways that align with the team.

### What he's actually doing (verified from git history + GitHub)

- **Downstream merges**: ~biweekly, manually resolving conflicts. Flow is healthy (PR #3159 merged Apr 29, #3168 in CI).
- **OTE test management**: Classifying upstream tests as informing/blocking. Informing tests deliberately disabled by jcaamano ([PR #3118](https://github.com/openshift/ovn-kubernetes/pull/3118)) — internal decision, not a gap for us.
- **CI flake fighting**: REAL and ongoing. On Apr 29 alone, triggered 11 jobs on one PR. His solution: personal `jluhrsen/pr-ci-dashboard` tool (active as of Apr 16).
- **Branch sync automation**: Building jobs in openshift/release for OCP 5.0 transition. His PRs need dptp team approval — not something external contributors can shortcut.

### Realistic collaboration approach

The original 5 items were overstated — none are realistic entry points as described. What IS realistic:
- [ ] **Build a fresh ci:pr-retest ai-helpers command** — Not rebasing PR #177 (stale, design criticism, Jamo moved on). Instead, build new command that uses AI reasoning (addressing reviewer feedback) on top of existing ci plugin's 25 commands. Would complement his pr-ci-dashboard.
- [ ] **Contribute upstream ovnk fixes** — The verified first PRs (Alpine, permissions, paths-ignore) don't need Jamo's involvement but demonstrate competence to the broader team he works with.
- [ ] **Talk to Jamo directly** — He's a former Submariner coworker. Asking "what would actually help?" is more effective than guessing from git history.

---

## Dropped After Verification

- ~~Upload-artifact v3→v7~~ — FALSE POSITIVE. `upload-pages-artifact@v3` is a different action.
- ~~.gitignore sensitive patterns~~ — No peer projects, no tracked sensitive files.
- ~~Shell script quoting (3 items)~~ — PEDANTIC. Dev scripts, hardcoded values.
- ~~Makefile .PHONY~~ — PEDANTIC. Zero functional impact.
- ~~Dockerfile.utest yum→dnf~~ — yum=dnf symlink on CentOS 9. File unused in CI.
- ~~ovnk-mcp lint.sh ${HOME}~~ — $HOME never has spaces.
- ~~network-tools Go 1.22.0~~ — ART-managed repo.
- ~~libovsdb curl|sh~~ — Standard golangci-lint practice.
- ~~SECURITY.md for 8 repos~~ — 4 covered by org policy, 4 upstream.
- ~~docker.yml paths-ignore~~ — Push-only, near-zero benefit.
- ~~retention-days~~ — Count was wrong, free storage.
- ~~fixup!/squash! rejection~~ — Zero instances in last 200 commits. Not a problem.
- ~~Commit format/size enforcement~~ — Too cultural/invasive.
- ~~gofumpt~~ — Would reformat many files.
- ~~Shared lint framework~~ — Different project structure.
- ~~Help resolve merge PR #3142~~ — Bot-generated branch sync (master→release-4.22), not a stuck downstream merge. On hold by design, needs release context.
- ~~Automate OTE test annotation diffs~~ — ALREADY AUTOMATED. `downstream-sync-commands.sh` runs `update-tests-annotation.sh` automatically. Only failure path is manual.
- ~~Help debug CNO PR #2968~~ — Only 6 days old, needs maintainer review (mattedallo/tssurya). External help limited to lint failure diagnosis. Not a strong collaboration opportunity.
- ~~Branch sync PR #76204~~ — Practically dead (lifecycle/rotten). Design disagreement on removing `-X theirs`. No consensus.
- ~~Pick up ai-helpers PR #177 as-is~~ — Stale 5 months, design criticized by reviewers (bentito: "not sure how much AI this is"), Jamo moved to personal `pr-ci-dashboard`. Resurrecting it would be more work than starting fresh.
- ~~OTE informing test presubmit job~~ — Deliberately disabled by jcaamano (PR #3118, Apr 7). Internal decision, not our call to re-enable.
- ~~Help land branch sync PR #77875~~ — Rehearsal test failing, needs dptp approval. We have no authority in openshift/release.
- ~~CI config templating~~ — Solved problem. config-brancher + prowgen already handle per-branch configs by design. Duplication is intentional.

---

## Verified SHA Inventory (for pinning PR)

| Action | Tag | Commit SHA |
|--------|-----|------------|
| actions/checkout | @v6 | de0fac2e4500dabe0009e67214ff5f5447ce83dd |
| actions/upload-artifact | @v7 | 043fb46d1a93c77aae656e7c1c64a875d1fc6a0a |
| actions/setup-go | @v6 | 4a3601121dd01d1626a1e23e37211e3254c1c06c |
| actions/download-artifact | @v8 | 3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c |
| actions/cache | @v4 | 0057852bfaa89a56745cba8c7296529d2fc39830 |
| actions/setup-python | @v5 | a26af69be951a213d495a4c3e4e4022e16d87065 |
| actions/labeler | @v5 | 8558fd74291d67161a8a78ce36a881fa63b766a9 |
| actions/stale | @v9 | 5bef64f19d7facfb25b37b414482c7164d639639 |
| actions/configure-pages | @v5 | 983d7736d9b0ae728b81ab479565c72886d7745b |
| actions/upload-pages-artifact | @v3 | 56afc609e74202658d3ffba0e8f6dda462b719fa |
| actions/deploy-pages | @v4 | d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e |
| docker/setup-buildx-action | @v3 | 8d2750c68a42422c14e847fe6c8ac0403b4cbd6f |
| docker/metadata-action | @v5 | c299e40c65443455700f0fdfc63efafe5b349051 |
| docker/login-action | @v3 | c94ce9fb468520275223c153574b00df6fe4bcc9 |
| docker/build-push-action | @v5 | ca052bb54ab0790a636c9b5f226502c73d547a25 |
| golangci/golangci-lint-action | @v8.0.0 | 4afd733a84b1f43292c63897423277bb7f4313a9 |
| apache/skywalking-eyes | @v0.8.0 | 61275cc80d0798a405cb070f7d3a8aaf7cf2c2c1 |

Note: golangci-lint-action and skywalking-eyes have annotated tags — SHAs above are commit SHAs.

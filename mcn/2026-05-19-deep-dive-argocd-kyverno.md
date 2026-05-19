---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, argocd, kyverno, harden-runner, renovate, slsa]
---

# Deep Dive: Argo CD and Kyverno Repo Automation

File-level deep dives of argoproj/argo-cd (20+ files) and
kyverno/kyverno (24 files). Both have mature, production-ready
CI/CD that complements patterns from Cilium/Istio/etcd.

## Argo CD — Key Findings

### 1. Harden-Runner with Graduated Policy

Every workflow uses `step-security/harden-runner@v2.19.3`. Two
modes:

- **Audit mode**: CI, release, CodeQL, image build — observe but
  don't block
- **Block mode with allowlists**: stale (only `api.github.com`),
  Renovate (Go proxy, npm, Helm, GHCR), Snyk (AWS ECR CloudFront,
  Quay, Docker Hub)

Global kill switch: `if: ${{ vars.disable_harden_runner != 'true' }}`
lets forks easily disable.

**MCN**: Start with audit mode everywhere. Tighten to block mode
for simple jobs (stale, labeler). The allowlist from Snyk workflow
is a good reference for container scanning endpoints.

### 2. Renovate — Disable-All-Then-Enable Pattern

Modular presets in `renovate-presets/` directory:

- `fix/disable-all-updates.json5`: `matchPackageNames: ["*"],
  enabled: false` — disables everything by default
- Individual presets re-enable specific things
- Coexists with Dependabot (Dependabot handles GHA + Go modules,
  Renovate handles tool versions via custom regex managers)

Custom regex managers detect `# renovate: datasource=... packageName=...`
comments in YAML and shell files, letting Renovate update inline
version pins.

Post-upgrade command: `make mockgen` after mockery tool updates.

OSV vulnerability alerts + merge confidence badges in PR bodies.

**MCN**: The disable-all pattern prevents surprise updates. The
`# renovate:` comment convention is excellent for managing version
pins in workflow files and scripts.

### 3. Triple SLSA Provenance

Release pipeline generates three separate SLSA Level 3 provenance
attestations:

1. Container images (via `generator_container_slsa3`)
2. CLI binaries (via `generator_generic_slsa3`)
3. SBOMs (via `generator_generic_slsa3`)

Plus cosign keyless signing of all images. Plus SPDX SBOMs for Go
deps, npm deps, and Docker image layers.

**MCN**: Gold standard for supply chain security. Adopt at least
image signing + provenance for releases.

### 4. GODEBUG Security Flags

All Go build commands include:
`GODEBUG="tarinsecurepath=0,zipinsecurepath=0"`

Prevents path traversal in tar/zip operations. Simple, zero-cost
hardening.

**MCN**: Add to all Go build commands immediately.

### 5. gotestsum with Rerun-Fails

Uses `gotestsum --rerun-fails` with configurable retry count
(`ARGOCD_E2E_RERUN_FAILS=5`). Reduces flaky test noise without
masking real failures.

**MCN**: Better than manual reruns. Adopt once tests exist.

### 6. Cherry-Pick Label Automation

Adding `cherry-pick/<version>` label to merged PR triggers
automatic cherry-pick. Matrix strategy fans out for multiple
labels. Uses GitHub App token (not GITHUB_TOKEN) so cherry-pick
PRs trigger CI. Comments on original PR with link to cherry-pick.

**MCN**: Clean, production-ready. Adopt once release branches
exist.

### 7. PR Title Conventional Commit Enforcement

`thehanimo/pr-title-checker` enforces:
`^(refactor|feat|fix|docs|test|ci|chore|revert)!?(\(.*\))?!?:.*`

Adds `title needs formatting` label on failure. Enables automated
changelogs via GoReleaser.

**MCN**: Pairs with release-please for automated versioning.

### 8. Composite Result Pattern for Matrix CI

A `test-e2e-composite-result` job aggregates matrix results into
a single status check, treating `skipped` as success. Solves
GitHub's problem of requiring specific matrix job names as status
checks.

**MCN**: Immediately useful for any matrix CI.

### 9. Snyk Report as Documentation

Weekly Snyk scan generates a report, creates PR to add it to the
docs directory. Published security posture, not just a CI artifact.

**MCN**: Increases transparency. Consider publishing vulnerability
posture alongside release notes.

## Kyverno — Key Findings

### 10. Workflow Failure Issue Tracker

`.github/actions/workflow/failure-issue/action.yaml`

Composite action that auto-creates/closes GitHub issues on CI
failure. Used in every workflow via `sync-issue` job.

- Unique ID per workflow+branch embedded as HTML comment:
  `<!-- workflow-failure:Trivy:refs/heads/main -->`
- On failure: creates or updates issue with metadata table
  (run ID, branch, commit, actor, job results)
- On success: finds matching issue and closes it
- Pattern: `if: always() && github.event_name == 'push'` — only
  tracks failures on push (not PRs)

**MCN**: Best CI failure tracking pattern found in any project.
Single composite action turns broken CI into trackable issues.

### 11. k6 Performance Testing with KWOK

Performance pipeline:

1. Create KIND cluster with Prometheus
2. Install KWOK (Kubernetes Without Kubelet) — fake node with
   32 CPU / 256Gi, no real compute needed
3. Run k6 with 100 virtual users, 1000 iterations
4. k6 writes metrics to Prometheus via remote write
5. Post-test: query Prometheus for etcd metrics, pod CPU/memory

Three scenarios compared: bare K8s vs Kyverno vs Kyverno+policies.
Directly measures admission controller overhead.

**MCN**: The KWOK pattern is clever — create many K8s objects
without needing real workload execution. Good for measuring
operator reconciliation overhead.

### 12. Trivy-to-GitHub-Issues Bidirectional Sync

Three-layer pipeline:

1. `scan-trivy.yaml`: scans on push, uploads SARIF
2. `periodic-trivy.yaml`: daily cron triggers scans across
   main + release branches
3. `sync-trivy-issues.yaml`: creates issues from alerts,
   auto-closes when alerts resolve

Deduplication via `<!-- codeql-id-NNN -->` HTML comments. Multi-
branch aggregation lists affected branches per finding.

**MCN**: Directly applicable. Auto-create tracked issues from
vulnerability findings. Auto-close when fixed. Multi-branch
coverage.

### 13. Mandatory Milestone Check

12 lines of JavaScript in GitHub Actions:

```javascript
if (data.milestone) {
  core.info(`Milestone set: ${data.milestone.title}`);
} else {
  core.setFailed(`A milestone needs to be set.`);
}
```

Triggers on: opened, milestoned, demilestoned, edited, synchronize.
The `demilestoned` trigger re-fails if someone removes the milestone.

**MCN**: Forces every PR to be associated with a release version.
Essential for accurate release changelogs. Trivially simple.

### 14. SLSA Level 3 for All Artifacts

7 parallel SLSA provenance jobs — one per container image.
CycloneDX SBOMs for each image. Cosign signing of images and
OCI install manifests. Krew plugin distribution for kubectl.

**MCN**: Kyverno's per-image SLSA provenance is the most
granular approach found.

### 15. Stale Branch Cleanup

`cbrgm/cleanup-stale-branches-action` daily at midnight. Deletes
branches matching allowed prefixes (dependabot, cherry-pick) with
no commits in 7 days.

**MCN**: Simple hygiene. Adopt from day one.

### 16. PR Branch Auto-Updater

Keeps all open PR branches up to date with base branch
automatically. Uses bot token with `workflow` write permission
(needed for PRs touching workflow files).

**MCN**: Reduces merge conflict churn for active repos.

## Top MCN Takeaways (Combined)

1. **Workflow failure issue tracker** (Kyverno) — auto-creates
   issues for broken CI, auto-closes when fixed
2. **Harden-runner graduated policy** (Argo CD) — audit for complex
   builds, block for simple jobs, fork toggle
3. **GODEBUG security flags** (Argo CD) — zero-cost build hardening
4. **Mandatory milestone check** (Kyverno) — 12 lines, huge impact
   on release management
5. **Triple SLSA provenance** (Argo CD) — images, binaries, SBOMs
6. **Trivy-to-issues bidirectional sync** (Kyverno) — auto-track
   and auto-close vulnerability findings
7. **Renovate disable-all-then-enable** (Argo CD) — prevent
   surprise updates, coexist with Dependabot
8. **Cherry-pick label automation** (Argo CD) — matrix fan-out for
   multi-branch cherry-picks
9. **k6 + KWOK perf testing** (Kyverno) — measure operator overhead
   without real compute
10. **Composite result pattern** (Argo CD) — aggregate matrix CI
    into single status check

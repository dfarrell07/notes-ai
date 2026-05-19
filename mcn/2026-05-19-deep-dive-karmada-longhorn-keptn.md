---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, karmada, longhorn, keptn]
---

# Deep Dive: Karmada, Longhorn, Keptn

File-level deep dives. Keptn (15+ files) completed. Karmada and
Longhorn results pending.

## Keptn — Key Findings

### 1. GitHub Projects V2 Date Auto-Population

File: `.github/workflows/set-date.yml`

On issue assignment: sets "Start Date" to today. On issue close:
sets "End Date". Uses raw `gh api graphql` mutations against
Projects V2 API. Zero-overhead time tracking.

**MCN**: If using GitHub Projects for sprint tracking,
auto-populate dates for accurate cycle-time metrics.

### 2. Kube-burner for Controller Scalability

Files: `test/load/cfg.yml`, `metrics.yml`

Creates 100 `KeptnMetric` CRs at controlled QPS (4 req/s, burst
4). Scrapes Prometheus for operator memory, CPU, throttling.
Measures controller scalability, not cluster scalability.

**MCN**: Use kube-burner to measure MCN controller reconciliation
performance under load.

### 3. release-please Monorepo with Per-Component Config

Files: `release-please-config.json`,
`.release-please-manifest.json`

6 independently versioned packages. Each uses `monorepo-tags` for
tag namespacing, `exclude-paths` to avoid cross-triggering, and
`extra-files` for version propagation (Makefile, Chart.yaml).
`x-release-please-version` comment markers enable in-place
replacement.

**MCN**: If MCN ever has multiple components in one repo,
release-please monorepo support is mature.

### 4. Two-Tier Stale Bot — Good First Issue Pipeline

File: `.github/workflows/stale.yml`

Standard 60-day stale + special handling for "good first issue":
assigned but inactive issues get `update-requested` at 21 days,
`to be unassigned` at 28 days. Creates a pipeline for reclaiming
reserved issues.

**MCN**: Community management pattern for contributor-reserved
issues that aren't being worked on.

### 5. Security Scan with Artifact Reuse

File: `.github/workflows/security-scans.yml`

Weekly scan downloads CI artifacts from last successful run
(queries GitHub API for run ID). Runs 4 scanners in parallel:
KICS (IaC), kubeconform, Kubescape (NSA/MITRE frameworks),
Trivy. Auto-creates issue on failure.

**MCN**: Efficient — reuses existing artifacts instead of
rebuilding. Four parallel scanners give comprehensive coverage.

### 6. Chainsaw (Kyverno) as Integration Test Framework

Files: `.chainsaw.yaml`, `test/chainsaw/*/chainsaw-test.yaml`

Instead of Ginkgo, uses Kyverno's Chainsaw with Go templating.
Numbered YAML files (00-install.yaml, 00-assert.yaml) as
sequential test steps. `skipDelete: true` for debugging.

**MCN**: Alternative to Ginkgo for YAML-heavy integration tests.
Worth evaluating for operator deployment validation.

### 7. SECURITY-INSIGHTS.yml (OpenSSF Standard)

File: `SECURITY-INSIGHTS.yml`

Machine-readable file declaring project security posture:
vulnerability reporting contacts, release process URLs,
distribution points, maintainer references.

**MCN**: Emerging CNCF/OpenSSF standard. Add from day one.

### 8. controller-gen Inside Docker Build

File: `lifecycle-operator/Dockerfile`

Runs `controller-gen object:headerFile=...` inside the builder
stage before compiling. Generated deepcopy code is always fresh.
Eliminates "forgot to run code-gen" bugs.

**MCN**: Consider running code generation inside Docker build
for guaranteed freshness.

### 9. Renovate Custom Regex for Makefile/YAML Versions

File: `renovate.json`

Custom regex manager finds `# renovate: datasource=... depName=...`
comments followed by `_VERSION` variable assignments. Works across
Makefiles, Dockerfiles, YAML, TOML, shell scripts.

Combined with `minimumReleaseAge: 3 days` for stability.

**MCN**: The `# renovate:` comment convention lets Renovate update
version pins in any file format.

### 10. Shared golangci-lint Config Across Sub-Modules

File: `.golangci.yml` (repo root)

One config shared by all 3 Go modules. Each runs with
`--config ../.golangci.yml`. Enables uncommon linters:
`containedctx`, `nilnil`, `noctx`.

**MCN**: If MCN has multiple Go modules, share one lint config.

### 11. OCI Image Tarballs as Cross-Job Artifacts

File: `.github/workflows/CI.yaml`

Builds Docker images, exports as OCI tarballs
(`outputs: type=oci,dest=/tmp/...tar`), uploads as artifacts.
Downstream jobs download and load. Tag communicated via separate
artifact file.

**MCN**: Same pattern as Thanos gotesplit — build once, share
across parallel test jobs without a registry.

## Longhorn — Key Findings

### 12. Sprint Automation via Projects V2 GraphQL

Files: `.github/workflows/periodic-issue-sprint-update.yml`,
`check-sprint-last-day.py`

Python script calculates sprint boundaries (14-day sprints) via
GraphQL `ProjectV2IterationField`. Manages 3 project boards with
status-dependent rollover rules (Review items advance, others
clear).

**MCN**: Most sophisticated sprint automation found. If using
GitHub Projects for sprint tracking, adopt the status-aware
rollover pattern.

### 13. Org Membership Routes Community vs Internal Issues

File: `.github/workflows/update-community-issue.yml`

Uses `tspascoal/get-user-teams-membership` to check if issue
author is org member. Non-members go to Community Sprint board;
members go to Internal Sprint board. Comment activity from
community members auto-transitions issues to "In Progress".

**MCN**: Clean community engagement tracking. Separates community
and internal work streams automatically.

### 14. Label-Triggered Sibling Issue Creation

File: `.github/workflows/create-issue.yml`

Three label triggers:

- `backport/X.Y` -> creates `[BACKPORT][vX.Y]` sibling
- `require/auto-e2e-test` -> creates `[TEST]` sibling on QA board
- `require/ui` -> creates `[UI]` sibling

Removing the label closes the sibling. Bidirectional lifecycle.

**MCN**: Enforces workflow chains — filing a feature auto-generates
test tracking.

### 15. PR Review Reminder with Slack User Mapping

File: `.github/workflows/pr-review-reminder.yml`,
`pr-review-reminder.py`

Scans 12 repos for open PRs, converts GitHub usernames to Slack
IDs via mapping, sends @mention notifications. Runs weekly.

**MCN**: Multi-repo review reminder with identity mapping.

### 16. Release Task Issue Generator

File: `.github/workflows/create-release-task-issues.yml`

Takes version, release captain, QA captain as inputs. Creates:

- Always: main release task + CVE fix tracking
- Feature releases (.0 only): regular tasks + perf benchmarks

Conditional issue generation based on version parsing.

### 17. "Won't Fix" Auto-Label on Not-Planned Close

File: `.github/workflows/wont-fix.yml`

Maps GitHub's native `state_reason === "not_planned"` close reason
to project-specific `wontfix` label and clears milestone. First
project surveyed mapping close-reason metadata to labels.

### 18. LEP (Enhancement Proposal) Process

Directory: `enhancements/` (90+ proposals)

Structured template: Summary, Motivation, Goals, User Stories,
API Changes, Design, Test Plan, Upgrade Strategy. Features tagged
`require/lep` must have a corresponding proposal.

**MCN**: Lightweight version of Kubernetes KEPs. Worth adopting
for MCN architectural decisions.

## Karmada — Key Findings

### 19. PR Performance Comparison with Matplotlib

Files: `.github/workflows/ci-performance-compare.yaml`,
`hack/performance/collect-metrics.sh` (836 lines),
`hack/performance/visualize-metrics.py`

`workflow_dispatch` with `target_pr_number` input. Runs ClusterLoader2
on base AND PR in parallel. Collects 10 Prometheus metric categories
(P50/P90/P99 latencies, throughput, queue depths, CPU/memory).
Matplotlib generates baseline (dashed) vs target (solid) charts.

**MCN**: Performance regression detection before merge with visual
diffs. Best perf comparison pattern found.

### 20. 40-Combination API Compatibility Matrix

File: `.github/workflows/ci-schedule-compatibility.yaml`

10 K8s API server versions (1.26-1.35) x 4 Karmada versions
(master + 3 release branches) = 40 combinations. Weekend-only
schedule. `max-parallel: 5` throttle.

**MCN**: Widest K8s compatibility matrix found. For multi-cluster
projects, test control plane against member cluster API versions.

### 21. CRD Survival Test After Helm Uninstall

File: `.github/workflows/installation-chart.yaml`

After `helm uninstall`, verifies CRD still exists — prevents
accidental cascade deletion of CRDs and their data.

**MCN**: Critical test for any Helm-distributed operator.

### 22. Three Installation Methods, Each with K8s Matrix

Files: `installation-chart.yaml`, `installation-cli.yaml`,
`installation-operator.yaml`

Helm, CLI, and Operator installation each tested against 3 K8s
versions. Different E2E entry points per method.

**MCN**: If MCN supports multiple installation methods, test
each independently.

### 23. Gemini AI Review with Secure Coding Style Guide

Files: `.gemini/config.yaml`, `.gemini/styleguide.md`

Custom style guide includes: interface compliance checks, max 5
function parameters, prohibit hardcoded credentials and custom
cryptography, changelog formatting rules.

**MCN**: First project seen with secure coding rules in an AI
review style guide. Worth including security rules in MCN's
CLAUDE.md.

### 24. SECURITY-INSIGHTS.yml

File: `SECURITY-INSIGHTS.yml`

OSSF standard for machine-processable security metadata. Same
pattern as Keptn. Declares tools (Dependabot, Trivy), contacts,
SBOM references, dependencies policy.

**MCN**: Emerging standard — adopt alongside OSSF Scorecard.

### 25. 4 Separate SLSA Provenance Attestations

File: `.github/workflows/release.yml`

Separate provenance for: CLI binaries, CRDs, Helm charts, SBOMs.
Most granular SLSA implementation found.

### 26. golangci-lint v2 with modernize + depguard

File: `.golangci.yml`

`modernize` linter with `omitzero` disabled (would change
metav1.ObjectMeta tags). depguard blocks `gopkg.in/yaml.v3`
(archived April 2025, use `sigs.k8s.io/yaml`).

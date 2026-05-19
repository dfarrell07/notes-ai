---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, cilium, automation, renovate, cosign, ariane]
---

# Deep Dive: Cilium Repo Automation

Actual file-level deep dive of github.com/cilium/cilium. 25+ files
read. Cilium is the most heavily automated Go networking project in
the K8s ecosystem — 93 workflow files, 42 composite actions, 1136-line
Renovate config.

## Key Findings for MCN

### 1. Ariane — Comment-Driven Test Orchestration

File: `.github/ariane-config.yaml`

Custom system mapping PR comment triggers to workflows with DAG-style
`depends-on` dependencies. Example: typing `/test` triggers 20+
conformance workflows, all depending on `/build-images-dependency`.

Features:

- Regex capture groups as parameters: `/ci-gke versions=all`
- Per-workflow `paths-ignore-regex` for smart skipping
- Nightly schedule section with per-branch time offsets
- `allowed-teams: [organization-members]` for access control

Scheduled runner (`.github/workflows/ariane-scheduled.yaml`): hourly
cron, but each branch triggers every 6 hours using modular arithmetic
to avoid simultaneous runs.

**MCN**: The `depends-on` DAG pattern for "build images before tests"
is directly useful. Full Ariane is heavy infrastructure — a simplified
version using `workflow_dispatch` + comment parsing would work.

### 2. Renovate Config — 1136 Lines of Gold

File: `.github/renovate.json5`

Most sophisticated Renovate config in any OSS project surveyed:

- Self-hosted with dedicated bot account
- 65+ included paths explicitly listed
- Multi-branch: main, v1.19, v1.18, v1.17
- 5-day minimum release age cooldown
- ~50 package name patterns auto-merged with only ciliumbot review
- Post-upgrade tasks: `go mod tidy`, `go mod vendor`,
  `make generate-apis`, `make manifests`,
  `make -C Documentation update-helm-values`
- Custom version regex for non-standard patterns (LVH images,
  envoy builds, image-tools timestamps)
- Per-branch version pinning: Alpine < 3.24 on v1.19, < 3.23 on
  v1.18; Go < 1.26 on stable branches
- Bug report template version dropdown auto-updated by Renovate

Auto-approve workflow: separate `auto-approve` environment gates
automatic approval of Renovate PRs.

**MCN**: The auto-merge for trusted deps, post-upgrade make targets,
and multi-branch version pinning are directly applicable patterns
once MCN has release branches.

### 3. golangci-lint — Custom Rules Worth Adopting

Files: `.golangci.yaml`, `.custom-gcl.yaml`

Enabled linters: copyloopvar, depguard, err113, errorlint,
forbidigo, goheader, gomodguard, gosec, govet, ineffassign,
misspell, modernize, sloglint, staticcheck, testifylint, unused

Notable custom rules:

- `forbidigo`: Blocks ~40 specific netlink functions requiring use
  of a `safenetlink` wrapper (project-specific safety)
- `gomodguard`: Blocks 12 deprecated modules with recommendations:
  - `math/rand` -> `math/rand/v2`
  - `go.uber.org/multierr` -> `errors` (Go 1.20+)
  - `golang.org/x/exp/maps|slices` -> stdlib
  - `github.com/sirupsen/logrus` -> `log/slog`
  - `gopkg.in/yaml.v2` -> `go.yaml.in/yaml/v2`
  - `k8s.io/utils/pointer` -> `k8s.io/utils/ptr`
- `sloglint`: camelCase keys, no raw keys, no mixed args, forbidden
  keys (time, level, msg, source)
- kube-api-linter via `.custom-gcl.yaml`: enforces K8s API
  conventions on CRD types

3 custom lint tools as separate Go binaries:

- `metricslint` — validates Prometheus metrics conventions
- `cloud-dep-check` — prevents cloud-specific deps in generic pkgs
- `statedblint` — validates StateDB table patterns

**MCN**: Adopt gomodguard for blocking deprecated modules. Adopt
kube-api-linter. Consider sloglint if standardizing on slog.

### 4. Cosign Composite Action with Retry

File: `.github/actions/cosign/action.yaml`

Reusable action for image signing + SBOM:

1. Install cosign via `sigstore/cosign-installer`
2. Generate SBOM via `anchore/sbom-action` (SPDX format)
3. Sign image with cosign using OIDC (keyless)
4. Attach SBOM as in-toto attestation
5. Exponential backoff retry (5 attempts, starting at 10s)

**MCN**: Directly reusable composite action pattern.

### 5. Workflow Telemetry — 37 Workflows

`cilium/workflow-telemetry-action@v2.2.0` used in every real
workflow. Always first step, before checkout. Collects CPU, memory,
disk I/O, network metrics per step. `comment_on_pr: false` for
silent collection.

**MCN**: Low-effort, high-value. Add as first step in all workflows.

### 6. Release Pipeline — Step-Based Architecture

Files: `build-images-releases.yaml`, `release.yaml`

Pipeline:

1. Tag push triggers image build matrix (9 images, amd64+arm64)
2. Each image signed with cosign, SBOM attached as attestation
3. Digests written to `Makefile.digests` artifact
4. Post-release step calls external `cilium/release` Go tool
5. Helm chart PR auto-created, checks waited for, auto-merged
6. OCI Helm chart at quay.io signed with cosign

Step-based workflow: pick which step to run (2-prepare-release,
4-post-release, 5-publish-helm). Allows re-running individual
phases.

**MCN**: Step-based release is mature. Auto Helm chart PR merge
could adapt to operator bundle releases.

### 7. Disk Cleanup Action

File: `.github/actions/disk-cleanup/`

Aggressively cleans runner disk: deletes Android SDK, .NET, Chrome,
Edge, PowerShell, Haskell, Mono, Julia, Node modules — all in
parallel with `nice -n -20 ionice -c 1 -n 0` for max I/O priority.

**MCN**: Immediately useful — disk space issues on runners are
common.

### 8. Makefile — 12 Prechecks

File: `Makefile` (650 lines)

`precheck` target runs 12+ custom shell-script checks: format,
log newlines, lock usage, viper usage, time package usage, BPF
header guards, datapath config, FIPS-only checks, etc.

`check-permissions` verifies no files are accidentally executable.

**MCN**: The precheck pattern with multiple targeted checks is
worth adopting for MCN-specific invariants.

### 9. E2E — Declarative YAML Test Matrix

Files: `.github/actions/e2e/*.yaml`

Test configurations defined as structured YAML files:

- `lb.yaml` — 5 load balancer configs
- `misc.yaml` — 7 miscellaneous configs
- `ipsec.yaml`, `wireguard.yaml`, `netkit.yaml`
- `kernel-versions.yml` — pinned Linux kernel versions

Dimensions: kernel (5.15-6.18, rhel8.10), kube-proxy mode, tunnel
mode, LB mode, plus feature flags. Cilium tests across 6 different
Linux kernel versions — unusual in the K8s ecosystem.

**MCN**: Declarative test matrix YAML is excellent. Define MCN test
configs (BGP, EVPN, network segmentation) as similar YAML files.

### 10. Feature Status Tracking

Composite action runs `cilium features status` after every test,
capturing enabled features as markdown (GitHub Summary) and JSON
(artifact). Daily `feature-summary-report.yaml` aggregates across
all tests.

**MCN**: Prevents feature test coverage from silently degrading.
Worth building once MCN has multiple features.

### 11. Maintainers Little Helper

File: `.github/maintainers-little-helper.yaml`

Enforces:

- All commits must have `Signed-off-by:` (blocks with
  `dont-merge/needs-sign-off` label)
- All PRs must have a `release-note/*` label (blocks with
  `dont-merge/needs-release-note-label`)
- Any `dont-merge/*` label blocks merging

**MCN**: Adopt this pattern. Enforces sign-off and release note
labeling without requiring Prow.

### 12. PR Template — AI Disclosure

File: `.github/pull_request_template.md`

Requires AI Influence Level disclosure: "Write a short paragraph
that states whether you used machine learning models... and indicate
the rating using AI Influence Level (AIL)."

**MCN**: Forward-thinking. Consider adopting.

## Not Used by Cilium (Notable Absences)

- No step-security/harden-runner (relies on SHA pinning, minimal
  permissions, persist-credentials: false instead)
- No Dependabot (Renovate replaces it entirely)
- No CodeQL (relies on gosec via golangci-lint)
- No Prow (pure GitHub Actions + Ariane)

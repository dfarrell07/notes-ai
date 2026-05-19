---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, strimzi, envoy-gateway, falco]
---

# Deep Dive: Strimzi, Envoy Gateway, Falco

File-level deep dives. Strimzi (20+ files) completed. Envoy Gateway
and Falco results pending.

## Strimzi Kafka Operator — Key Findings

### 1. CVE Container Rebuild Without Source Rebuild

File: `.github/workflows/cve-rebuild.yml`

Most novel release pattern found across all deep dives:

1. At release time, compiled Java artifacts stored as OCI blobs in
   GHCR via ORAS (tagged `<version>-<buildRunId>`)
2. On CVE fix: pulls pre-built artifacts from ORAS
3. Rebuilds containers with updated base images only (4 arch)
4. Pushes with CVE suffix (e.g., `0.49.0-2`) first
5. **GitHub Environment approval gate** pauses workflow for human
   validation
6. After approval, pushes without suffix (replaces release image)

**MCN**: This decouples "source rebuild" from "base image update."
For CVEs in base image packages (glibc, openssl), you never need
to rebuild from source. The Environment approval gate is simpler
than any custom mechanism.

### 2. Data-Driven Test Pipeline Configuration

File: `pipelines.yaml` (separate from workflow YAML)

All test matrix permutations defined in a YAML config file, not
hardcoded in workflow YAML. A `generate-matrix` action reads the
config and outputs the GitHub Actions matrix. Supports parameters:
pipeline, profile, kafkaVersion, kubeVersion, architecture, RBAC
scope, install type, feature gates.

Comment-triggered: `/gha run pipeline=regression,upgrade kafkaVersion=4.1.0`

**MCN**: Far more maintainable than encoding test matrices in
workflow YAML. Separates test definitions from CI infrastructure.

### 3. GitHub Actions Unit Testing with act

Files: `.github/workflows/actions-tests.yml`,
`.github/tests/scenarios/*.yaml`,
`.github/tests/events/*.json`

Full unit test framework for composite actions using nektos/act:

- 19 parse-comment scenarios
- 7 generate-matrix scenarios
- 5 validate-matrix scenarios
- 4 perf-report scenarios

Each scenario: id, description, event type, fixture path, expected
outputs. CI iterates via yq, passes expectations as `EXPECT_*` env
vars, runs through `act`.

Also runs `actionlint` before the act tests.

**MCN**: Only project surveyed that unit tests its own GitHub
Actions. Critical for complex composite actions.

### 4. Single Version-Truth File

File: `kafka-versions.yaml`

One YAML file defines every Kafka version: download URLs, SHA512
checksums, third-party lib paths, `supported: true/false`. Drives
Docker builds, Helm charts, operator runtime, documentation, and
CI matrix — all from one file.

**MCN**: If MCN supports multiple OCP or K8s versions, a single
version-truth file eliminates drift.

### 5. ORAS for Pre-Built Artifact Storage

Release workflow stores compiled binaries as OCI artifacts in
GHCR via ORAS. The CVE rebuild workflow later pulls these instead
of rebuilding from source. Each artifact tagged
`<releaseVersion>-<sourceBuildRunId>` for traceability.

**MCN**: ORAS is increasingly the standard for non-container OCI
artifact storage. Good for build artifacts, SBOMs, configs.

### 6. Compare-Merge-Commit Action

File: `.github/actions/utils/compare-merge-commit/action.yml`

Before running system tests, checks if a Build workflow for the
same commit already ran. Downloads `commit-sha.txt` artifact,
compares merge SHAs. If they match, reuses build artifacts. If
not, triggers its own build. Waits up to 80 minutes for builds.

**MCN**: Prevents testing stale builds. Ensures system tests
always run against the latest commit.

### 7. ImportControl for Architecture Boundaries

File: `.checkstyle/checkstyle.xml`

Checkstyle's `ImportControl` module enforces which packages can
import from which — preventing cross-package dependencies (e.g.,
topic-operator cannot import cluster-operator internals).

**MCN**: Go equivalent: `depguard` in golangci-lint or
`import-boss` from code-generator.

## Falco — Key Findings

### 8. Two-Layer Engine Version Enforcement

Files: `.github/workflows/engine-version-weakcheck.yaml`,
`.github/workflows/ci.yml`

**Weak check**: When engine source files change but version header
doesn't, posts PR comment with `/hold` — advisory block.

**Strong check**: Build computes checksum by hashing driver schema
version, supported event types, and rule fields. Compares to
checksum stored in `falco_engine_version.h`. If checksums differ
but version not bumped, CI fails hard.

**MCN**: Two-layer version enforcement (advisory + hard) is the
most robust pattern found. For Go, hash exported API surfaces or
CRD schema to detect version-requiring changes.

### 9. Custom Semgrep Rules for Insecure APIs

Files: `.github/workflows/insecure-api.yaml`, `semgrep/*.yaml`

Pinned Semgrep container with `--baseline-commit` (only flags NEW
issues). 4 custom rules banning insecure C functions with CWE
references. Pattern exclusions for string literal args (known-safe).

**MCN**: The baseline-only scanning approach is key — no noise
from existing code. Custom rules enforce project-specific coding
standards as automated checks.

### 10. Format Patch Artifacts on Failure

File: `.github/workflows/format.yaml`

On formatting failure, generates and uploads a `.patch` file as
artifact so contributors can `git apply` the fix directly.

**MCN**: Nice DX improvement. Contributors don't need to run
formatting tools locally — just download and apply the patch.

### 11. Weekly Automated Dependency Bump PRs

File: `.github/workflows/bump-libs.yaml`

Weekly cron downloads latest upstream tarball, extracts commit
hash, computes SHA256, updates cmake files, auto-creates PR
with `--signoff`.

**MCN**: The weekly auto-PR pattern for tracking upstream
dependencies is widely useful.

### 12. Dual-Safeguard for Latest Tag

File: `.github/workflows/release.yaml`

Two checks before applying `latest` Docker tag:

1. GitHub API confirms this is the latest release
2. Version string has no pre-release suffix

Both must pass. Prevents accidental latest-tagging of RCs or
patch releases to older branches.

**MCN**: Defense-in-depth for Docker `latest` tag management.

## Top MCN Takeaways (Combined)

1. **CVE container rebuild** (Strimzi) — ORAS artifact storage +
   container-only rebuild + Environment approval gate
2. **Two-layer version enforcement** (Falco) — advisory /hold +
   hard checksum CI failure
3. **GitHub Actions unit testing** (Strimzi) — act + scenario YAML
   - fixture JSON
4. **Data-driven test pipelines** (Strimzi) — pipelines.yaml
   config drives matrix, not workflow YAML
5. **Format patch artifacts** (Falco) — upload fix as downloadable
   patch on formatting failure
6. **Custom Semgrep with baseline** (Falco) — project-specific
   rules, only flag new issues

## Envoy Gateway — Key Findings

### 13. Monthly Release Issue Auto-Creation from EOL Config

File: `.github/workflows/monthly-release-issue.yaml`

Cron runs 1st of each month. Reads version/EOL pairs from
`site/hugo.toml`. Filters out past-EOL versions. Creates one
issue per active branch from `.github/ISSUE_TEMPLATE/release.md`
(13-step checklist).

**MCN**: Directly adoptable. Maintain version/EOL pairs in config,
auto-create patch release tracking issues monthly.

### 14. Release Pre-Flight CI Gate

File: `.github/workflows/release.yaml`

Before releasing, verifies `build_and_test.yaml` succeeded for the
same commit SHA. Simple `gh run list` query prevents releasing
broken code.

**MCN**: Simple but effective gate. Add to release workflow.

### 15. E2E Test Timing Wrapper

File: `test/e2e/timing.go`

`WrapConformanceTestsWithTiming()` adds duration measurement to
every test. Configurable warning threshold via
`EG_E2E_WARN_DURATION` env var (default 1 min). Produces sorted
timing report at cleanup with PASS/FAIL/SKIP + duration.

**MCN**: Identifies slow tests that are optimization candidates.

### 16. Structured YAML Release Notes

File: `release-notes/v1.8.0.yaml`

Categories as structured data: breaking changes, security updates,
new features, bug fixes, performance improvements, deprecations.
Each is multi-line YAML string. Consumed by release workflow.

**MCN**: Better than freeform markdown for programmatic
consumption. Consider alongside Contour's per-PR changelog files.

### 17. Deploy Profile Matrix Testing

4 Helm values profiles (default, gateway-namespace-mode,
xds-name-scheme-v2, watch-namespaces) as separate conformance
matrix entries.

**MCN**: Test multiple deployment modes as separate matrix entries.
Catches mode-specific regressions.

### 18. go tool -modfile for Tool Isolation

All tools invoked via `go tool -modfile=tools/go.mod`. Cleaner
than `tools.go` import hacks. Keeps tool deps completely
separate from main module.

**MCN**: Modern Go pattern for tool isolation. Adopt over the
`tools.go` blank import approach.

### 19. PR Review AI Skill

File: `.agents/skills/pr-review/SKILL.md`

Claude Code skill definition for automated PR review with
structured checklist: API conventions, implementation changes,
feature coverage, release notes.

**MCN**: Create MCN-specific PR review skill alongside the
GHA-based AI review workflows.

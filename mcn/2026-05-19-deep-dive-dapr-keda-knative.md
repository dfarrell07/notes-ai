---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, dapr, keda, knative]
---

# Deep Dive: Dapr, KEDA, Knative Serving

File-level deep dives. Dapr (20+ files) and Knative (25+ files)
completed. KEDA results pending.

## Dapr — Key Findings

### 1. Version Skew Testing — Truly Unique

File: `.github/workflows/version-skew.yaml`

Bidirectional N-1 version skew matrix:

- `control-plane-master`: master control plane + previous sidecar
- `dapr-sidecar-master`: previous control plane + master sidecar

Integration tests swap binary paths via env vars. E2E tests swap
via Helm values.

The patch system is unique: `.github/scripts/version-skew-test-patches/`
maintains **56 per-release patches** that adapt older test assertions
to work against mixed-version deployments.

Triggerable via bot: `/test-version-skew 1.10.5`

**MCN**: If MCN has control plane + data plane components, this
bidirectional version skew testing is essential. The patch directory
pattern handles real-world test drift.

### 2. Binary Regression Testing with scipy

File: `.github/scripts/validate_sidecar_resources.py`

On every PR:

1. Builds binary from PR branch and baseline
2. Runs both, scrapes `go_memstats_heap_inuse_bytes` over 30 seconds
3. Welch's t-test (trimmed 20%) for statistical significance
4. Fails if p < 0.05 for memory increase
5. Also fails if binary size grows > 7MB

Uses Go heap-in-use instead of RSS because "RSS reflects OS reclaim
timing (MADV_FREE), so it's noisy across runs."

**MCN**: Unique in the entire K8s ecosystem. If MCN has resource-
sensitive components, gate PRs on statistical resource regression.

### 3. Cross-Repo SDK Testing

File: `.github/workflows/dapr-test-sdk.yml`

Tests runtime HEAD against 4 SDK repos (Python, Java, JS, Go).
Builds daprd from PR, replaces system binary, runs each SDK's
test suite. Runs every 12 hours on weekdays.

Java SDK tests include ToxiProxy for fault injection. Go SDK uses
Mechanical Markdown to validate documentation examples are runnable.

**MCN**: Cross-repo testing from server side is rare. Consider if
MCN has client libraries or CLIs.

### 4. Label-Triggered Cross-Repo Issue Creation

File: `.github/scripts/dapr_bot.js`

When `sdk-needed` label is applied, bot auto-creates tracking
issues in 6 SDK repos. When `docs-needed` is applied, creates
issue in docs repo.

**MCN**: Prevents cross-repo tasks from falling through cracks.

### 5. depguard with 18 Deny Rules

File: `.golangci.yml`

Most comprehensive dependency deny list found. Forces specific
package choices (e.g., `golang-jwt/jwt` banned for
`lestrrat-go/jwx/v2`). Uses `default: all` with 40+ disables.

### 6. Sidecar Flavor Builds

Makefile builds two binary flavors via Go build tags:

- `allcomponents` — includes everything
- `stablecomponents` — excludes alpha/beta

Separate Docker images and tags for each flavor.

**MCN**: If MCN has stable vs experimental features, consider
compile-time component selection.

### 7. go.mod Comment Integrity Check

File: `.github/scripts/check_go_mod.mjs`

Verifies `go.mod` still contains commented-out replace directives
for local development. Prevents accidental commit with uncommented
local paths.

**MCN**: Simple guard against a common mistake.

## Knative Serving — Key Findings

### 8. Centralized Workflow Syncing (Knobots)

Repo: `knative-extensions/knobots`

Push-based sync: physically copies workflow template files into
50+ downstream repos via automated PRs. `repos.yaml` lists all
repos with fork accounts and team assignees. `actions-omitted.yaml`
provides per-repo exclusions with glob prefix matching.

Three-level indirection: thin stub (synced by knobots) -> reusable
workflow (in knative/actions) -> composite actions. Go version set
in exactly one place for the entire organization.

**MCN**: Solves workflow drift for multi-repo orgs. A
`submariner-io/actions` repo would be the MCN equivalent.

### 9. Build-Once Registry Artifact for E2E

File: `.github/workflows/kind-e2e.yaml`

Build job:

1. Starts local Docker registry (`registry.local:5000`)
2. Builds all images with ko into local registry
3. Writes registry filesystem to disk (`~/artifacts/registry`)
4. Uploads as artifact

Test matrix (42 combinations): downloads artifact, mounts
pre-populated registry into KinD cluster. Images built ONCE,
shared across all 42 test jobs.

The `.local` domain trick avoids TLS: `go-containerregistry`
allows insecure registry for `*.local` hostnames.

**MCN**: Avoids rebuilding images N times for N test matrix
entries. Significant CI cost savings.

### 10. Vendored Shell Library via tools.go

File: `hack/tools.go`

```go
//go:build tools
import (
    _ "knative.dev/hack"
    _ "k8s.io/code-generator"
)
```

Blank imports force `go mod vendor` to pull shell scripts and
non-Go tools. Then `hack/update-deps.sh` sources
`vendor/knative.dev/hack/library.sh`.

**MCN**: Clean way to share shell library code across repos via
Go's vendoring. Avoids git submodules and runtime downloads.

### 11. ko as Build Tool AND YAML Generator

File: `hack/generate-yamls.sh`

`ko resolve` resolves Go import paths in K8s YAML, builds the
binary, packages in container, pushes to registry, replaces import
path with image reference. Single command builds AND deploys.

Per-dependency base image overrides in `.ko.yaml`:

```yaml
defaultBaseImage: ghcr.io/wolfi-dev/static:alpine
baseImageOverrides:
  github.com/tsenart/vegeta/v12: ubuntu:latest
```

**MCN**: If using ko for upstream builds, the per-dependency base
image override is useful for tool images.

### 12. Smart Style Checking with Path Filters

File: `reusable-style.yaml` in knative/actions

Uses `dorny/paths-filter` to detect changed file types.
Conditionally runs only relevant checks: Go lint for Go changes,
shellcheck for shell changes, yamlfixer for YAML changes,
actionlint for workflow changes.

**MCN**: Saves CI minutes by skipping irrelevant checks.

### 13. gomodguard — errors.Join Migration

File: `.golangci.yaml`

Blocks `go-multierror` and `go.uber.org/multierr` with
recommendation to use `errors.Join`. Blocks `ghodss/yaml` for
`sigs.k8s.io/yaml`. Build tags `e2e,hpa,upgrade` ensure test
code gets linted too.

## Top MCN Takeaways (Combined)

1. **Version skew testing with patch dirs** (Dapr) — bidirectional
   N-1 compatibility with real test drift handling
2. **Build-once registry artifact** (Knative) — build images once,
   share across 42 test matrix entries
3. **scipy binary regression** (Dapr) — statistical hypothesis
   testing for resource consumption in CI
4. **Centralized workflow sync** (Knative) — push templates to all
   repos, single source of truth for Go version
5. **Vendored shell library** (Knative) — share scripts via Go
   vendoring, no submodules
6. **Cross-repo issue creation** (Dapr) — auto-create tracking
   issues in dependent repos via labels

## KEDA — Key Findings

### 14. Three-Workflow E2E Gating System

Files: `pr-e2e-creator.yml`, `pr-e2e.yml`, `pr-e2e-checker.yml`

Most robust comment-triggered E2E implementation found:

- **Creator**: on PR open, creates placeholder Check Runs in
  "queued" state that block merge
- **Executor**: on `/run-e2e` comment, verifies commenter belongs
  to `keda-e2e-test-executors` team, then runs tests. Supports
  regex filtering: `/run-e2e TestKafka.*`
- **Checker**: on `skip-e2e` label, marks checks as success with
  "skipped by maintainer" output

Comment reaction feedback: success gets +1, failure gets -1.
Comment edited to include link to running workflow.

**MCN**: Most polished E2E gating pattern. The three-workflow
split (creator/executor/checker) is significantly more robust
than a single workflow.

### 15. Auto-Reviewer Separate from CODEOWNERS

Files: `.github/workflows/pr-notify.yml`, `.github/reviewers.yml`

CODEOWNERS gates required reviews. Auto-assign action adds
reviewers as notification. Vendor-specific teams get notified
about changes to their scaler code.

**MCN**: Split notification (auto-assign) from approval
(CODEOWNERS). Lets component teams know without being blockers.

### 16. depguard — Ban sync/atomic

File: `.golangci.yml`

Blocks `sync/atomic` with message "use type-safe atomics from
go.uber.org/atomic". File-level exceptions for test code.

**MCN**: Good pattern for enforcing API consistency — ban the
raw package, require the typed wrapper.

### 17. Dual Renovate + Dependabot

Renovate handles GitHub Actions SHA pinning via
`helpers:pinGitHubActionDigests`. Dependabot handles version
bumps for Go modules and Docker. Intentionally complementary.

**MCN**: If adopting both, deduplicate responsibility clearly.

### 18. Double Image Tagging on Main

File: `.github/workflows/main-build.yml`

Every push to main publishes images tagged both `main` AND the
full commit SHA. Both Cosign-signed. Every commit independently
addressable and verified.

**MCN**: Good for debugging — any commit on main can be precisely
referenced.

### 19. Pre-commit Inclusive Language Hook

File: `.pre-commit-config.yaml`

```yaml
entry: "(?i)(black|white)[_-]?(list|List)"
description: Use "deny_list" or "allow_list" instead.
```

Zero-dependency pygrep pattern. Also enforces alphabetical
sorting of scalers in registration file.

### 20. Raw GraphQL for Project Board Automation

File: `.github/workflows/auto-add-issues-to-project.yml`

Direct `gh api graphql` mutations to add issues to Projects V2.
No third-party action dependency. Simpler and more maintainable.

### 21. Custom CI Container

`ghcr.io/kedacore/keda-tools:1.26.2` pre-bakes all build
dependencies. Used across most workflows to avoid tool
installation time.

**MCN**: Consider a pre-built CI container if tool installation
becomes a CI bottleneck.

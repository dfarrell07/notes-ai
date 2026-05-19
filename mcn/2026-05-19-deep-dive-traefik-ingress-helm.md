---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, traefik, ingress-nginx, helm]
---

# Deep Dive: Traefik, Ingress-NGINX, Helm

File-level deep dives. Traefik (20+ files) completed.
Ingress-NGINX and Helm results pending.

## Traefik — Key Findings

### 1. Dynamic Unit Test Matrix from Go Code

Files: `internal/testsci/genmatrix.go`,
`.github/workflows/test-unit.yaml`

Go program uses `golang.org/x/tools/go/packages` to enumerate
all packages, splits into 12 groups, emits JSON matrix for
GitHub Actions. Auto-parallelizes without manual maintenance.

**MCN**: Strong pattern for large Go repos where package set
changes frequently.

### 2. go-test-split-action for Integration Test Parallelism

File: `.github/workflows/test-integration.yaml`

`hashicorp-forge/go-test-split-action@v2.0.1` splits test
functions across 12 runners. Drop-in solution — handles splitting
without manual per-test assignment.

**MCN**: Directly applicable to slow integration test suites.

### 3. safe-chain npm Package Age Gate

File: `.github/workflows/template-webui.yaml`

`@aikidosec/safe-chain` blocks npm packages published < 72 hours
ago. Mitigates publish-and-exploit supply chain attacks.

**MCN**: If MCN has any npm-based frontend or tooling, add this.

### 4. Daily Docker Image Sync with crane

File: `.github/workflows/sync-docker-images.yaml`

Daily cron diffs tag lists between Docker Hub and GHCR using
`crane ls`. Only copies missing tags. Efficient multi-registry
mirroring without rebuilds.

**MCN**: crane-based registry sync is far more efficient than
rebuilding per-registry.

### 5. golangci-lint "default: all" with Justified Disables

File: `.golangci.yml`

`default: all` enables every linter. ~30 explicitly disabled with
justification comments. Self-documenting and auditable.

Notable rules:

- `gomoddirectives.tool-forbidden: true` — forbids tool directives
- `tagalign` — enforces struct tag ordering
- `depguard` blocks `pkg/errors` -> use stdlib

**MCN**: Gold standard for lint config transparency.

### 6. Go-Templated GoReleaser Config per OS

Files: `internal/release/release.go`, `.goreleaser.yml.tmpl`

Go program generates per-OS goreleaser config from template.
Custom delimiters `[[ ]]` avoid conflict with GoReleaser's
`{{ }}`. 17-OS/arch matrix.

### 7. Testcontainers Replacing Docker Compose

File: `integration/integration_test.go`

Parses compose YAML in Go, creates containers via testcontainers-go.
Gives programmatic control without abandoning compose format.
Includes Docker Desktop detection with Tailscale VPN fallback.

**MCN**: Worth evaluating if MCN E2E tests currently shell out
to docker compose.

### 8. Gateway API + Knative Conformance in CI

Files: `test-gateway-api-conformance.yaml`,
`test-knative-conformance.yaml`

Runs official upstream conformance suites. Gateway API uses build
tags. Knative uses ko for local image builds.

**MCN**: Run conformance suites for any standard API MCN
implements.

## Ingress-NGINX — Key Findings

### 9. E2E Tests as In-Cluster Pods

Files: `test/e2e/run-e2e-suite.sh`, `test/e2e-image/e2e.sh`

Tests don't run from CI runner — they run as a pod inside the
cluster via `kubectl run --rm --attach`. JUnit reports extracted
via ConfigMap: test pod gzips XML, stores in ConfigMap, CI runner
retrieves afterward.

**MCN**: Guarantees test binary sees same network as controller.
Worth adopting for integration tests needing cluster-internal
visibility.

### 10. Dynamic Version Matrix for Vulnerability Scanning

File: `.github/workflows/vulnerability-scans.yaml`

Weekly cron dynamically discovers latest 3 release tags via
`git tag --list --sort=-version:refname`, builds JSON array,
feeds to matrix for Trivy scanning. Self-maintaining.

**MCN**: Vulnerability scanning that auto-discovers versions to
scan without manual updates.

### 11. zz-tmpl-* Reusable Workflow Convention

Files: `.github/workflows/zz-tmpl-k8s-e2e.yaml`,
`zz-tmpl-images.yaml`

`zz-` prefix sorts templates to bottom of file listings. Called
7 times from `images.yaml` with just `name: <image>`.

**MCN**: Clean naming convention for reusable vs entry-point
workflows.

### 12. 60+ gocritic Checks Individually Enabled

File: `.golangci.yml`

Most exhaustive gocritic configuration seen. Notable uncommon
checks: `badLock`, `syncMapLoadAndDelete`, `truncateCmp`,
`exitAfterDefer`.

### 13. K6 Performance with Kernel Tuning

File: `.github/workflows/perftest.yaml`

Tunes Linux kernel before K6 run:
`net.ipv4.ip_local_port_range`, `tcp_tw_reuse`.
Captures `vmstat` alongside test for system-level correlation.

### 14. Mage-Based Release Automation in Go

File: `magefiles/steps/release.go`

Type-safe, testable release orchestration. Pulls image digests
from k8s staging registry YAML (canonical source). Generates
release notes separating Dependabot from real changes.

## Helm — Key Findings

### 15. `.github/env` for Centralized Version Pinning

File: `.github/env` (2 lines)

```text
GOLANG_VERSION=1.26
GOLANGCI_LINT_VERSION=v2.11.3
```

All workflows: `cat ".github/env" >> "$GITHUB_ENV"`. Single source
of truth. One file change updates all CI jobs.

**MCN**: Eliminates version drift between workflows. Simpler than
per-workflow pinning.

### 16. `go mod tidy -diff` (Go 1.26+)

File: `.github/workflows/build-test.yml`

Single command replaces `go mod tidy && git diff --exit-code`.
Prints diff of what tidy would change, fails if differences exist.

**MCN**: Cleaner. Adopt once on Go 1.26+.

### 17. `-shuffle=on` Test Flag

File: Makefile

`TESTFLAGS := -shuffle=on -count=1` — randomizes test execution
order to catch hidden inter-test dependencies. `-count=1` disables
test caching.

**MCN**: Add to test flags from day one.

### 18. Golden File Testing with repeat Field

File: `internal/test/test.go`

186 golden files. `cmdTestCase.repeat` field runs previously-flaky
tests N+1 times to verify stability.

**MCN**: Cheap anti-flakiness pattern for specific tests.

### 19. AGENTS.md

File: `AGENTS.md`

Structured guidance for AI coding tools about codebase layout,
build commands, branching strategy, compatibility rules.

### 20. Canary Releases on Every Main Merge

File: `.github/workflows/release.yml`

Builds canary binaries on every push to main, uploads to Azure
Blob Storage with `overwrite: 'true'`. Always-available latest
binaries at predictable URL.

**MCN**: Consider canary builds for MCN operator/CLI.

### 21. Architecture-Conditional Race Detection

File: Makefile

Race detector disabled on s390x (not supported). Graceful
degradation to verbose-only mode.

**MCN**: If supporting non-amd64, handle race detector
unsupported platforms gracefully.

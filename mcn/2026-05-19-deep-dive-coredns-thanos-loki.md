---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, coredns, thanos, loki]
---

# Deep Dive: CoreDNS, Thanos, Loki

File-level deep dives. Thanos (23+ files) completed. CoreDNS and
Loki results pending.

## Thanos — Key Findings

### 1. faillint — Import Policy Enforcer

File: Makefile (`faillint` target)

Custom linter that bans specific import paths with redirects:

- `sync/atomic` -> `go.uber.org/atomic`
- `github.com/prometheus/tsdb` -> `prometheus/prometheus/tsdb`
- `io/ioutil` functions -> banned (deprecated)
- `fmt.Print/Println/Sprint` -> banned in non-test code

Stronger than `depguard` — catches individual function imports,
not just packages.

**MCN**: Adopt for enforcing library preferences. Ban deprecated
stdlib packages and redirect to blessed alternatives.

### 2. gotesplit — E2E Parallelization via Artifact Sharing

Files: `.github/workflows/go.yaml`, `Dockerfile.e2e-tests`

Build E2E image once with `-race` flag. `docker save | gzip` as
artifact. Fan out to 8 parallel runners via matrix. Each runner
downloads and `docker load`s the image.

`gotesplit` splits test list across `GH_INDEX` of `GH_PARALLEL`.

**MCN**: Clean alternative to pushing ephemeral CI images to a
registry. Avoids 8 redundant image builds.

### 3. Automated Base Image SHA Updater

Files: `.github/workflows/container-version.yaml`,
`scripts/busybox-updater.sh`, `.busybox-versions`

Hourly cron fetches latest busybox manifest digests per platform.
Writes per-arch SHA256s to `.busybox-versions`. Auto-creates PR.

**MCN**: Automate base image digest freshness. Replace manual UBI
image bumps.

### 4. Bingo — Version-Pinned Tool Management

File: `.bingo/Variables.mk`

Each tool gets its own `go.mod` file in `.bingo/`. Generated
Makefile variable includes exact version in binary name
(`golangci-lint-v2.4.0`). Multiple versions can coexist.

**MCN**: Cleaner tool management than ad-hoc `go install`. Tools
tracked in git with deterministic versions.

### 5. E2E with Race Detector Always On

File: `Dockerfile.e2e-tests`

E2E Docker image built with `-race` and `GORACE="halt_on_error=1"`.
Every E2E test catches concurrency bugs that unit tests miss.

**MCN**: Consider running E2E with race detector enabled, at
least in nightly CI.

### 6. Smart Crossbuild Branching

File: Makefile

`crossbuild` on `main` branch only builds 3 platforms (the ones
shipped as Docker images). On other branches, builds all 7. Saves
CI time on the critical path.

**MCN**: Reduce crossbuild scope on main to only shipped platforms.

### 7. Copyright Header as Go Program

File: `scripts/copyright/copyright.go`

Go program that walks the tree and auto-inserts correct copyright
headers. Distinguishes between project files and vendored files
(different headers).

**MCN**: More maintainable than shell-based header checks.

### 8. mdox Documentation Validation

Custom tool validates markdown docs, checks links, ensures
documentation flags match actual binary flags. `check-docs` builds
the binary, generates docs from it, verifies nothing changed.

**MCN**: If MCN has CLI docs generated from code, validate them
in CI.

### 9. require_clean_work_tree Macro

Makefile macro used extensively to fail CI if generated files are
not committed. Applied to `go-lint`, `check-docs`, `check-examples`.

**MCN**: Robust "nothing drifted" check.

## CoreDNS — Key Findings

### 10. Per-Plugin Fuzz Harness

Files: `plugin/pkg/fuzz/do.go`, 7 plugin fuzz targets

Shared `fuzz.Do()` unpacks DNS message, calls `plugin.ServeDNS()`.
Each plugin instantiates itself and delegates. The grpc fuzzer is
217 lines with 12 `fakeClient` behavior modes selected by first
byte of fuzz data.

**MCN**: Excellent pattern for any project with handler interface.

### 11. Plugin Ordering via plugin.cfg

File: `plugin.cfg`

Text file listing `name:package` pairs in execution order. Code
generation produces blank imports and ordered directive list.
`COREDNS_PLUGINS` env var adds plugins at build time.

### 12. Dependabot Cooldown

`cooldown: default-days: 7` prevents rapid-fire PRs after major
upstream releases.

## Loki — Key Findings

### 13. Two Claude Code Workflows

Files: `.github/workflows/claude-code-review.yml`,
`.github/workflows/claude.yml`

Label-triggered automated review (on `claude-review` label) +
interactive `@claude` mention-driven workflow. API key from
Grafana Vault. Extensive allowed_tools whitelist for interactive
mode.

**MCN**: Two-workflow pattern (automated + interactive) with
proper security boundaries.

### 14. Helm Diff CI with K3D Clusters

File: `.github/workflows/helm-diff-ci.yml`

7-scenario matrix. For K3D scenarios: installs chart, runs
`helm diff upgrade`. Summary job collects diffs, wraps in
collapsible `<details>` blocks, posts as sticky PR comment.
Truncated to 65535 chars.

**MCN**: Shows rendered K8s resource diffs for chart changes.
Catches unintended side effects invisible in template diffs.

### 15. Jsonnet-Generated Release Workflows

File: `.github/release-workflows.jsonnet`

Single Jsonnet file generates patch-release-pr, minor-release-pr,
release, check, and images workflows. Prevents drift between
workflow variants.

**MCN**: If MCN has multiple workflow variants (patch vs minor
vs major releases), generate from a single source.

### 16. Conventional Commit Enforcement on PR Titles

File: `.github/workflows/conventional-commits.yml`

Requires uppercase first letter after type prefix.

### 17. Renovate — Disable All on Release Branches

File: `.github/renovate.json5`

Release branches get all updates disabled except security via
`vulnerabilityAlerts`. Main branch auto-merges minor/patch Go
dependency updates.

**MCN**: Smart release branch policy — only security updates on
stable branches.

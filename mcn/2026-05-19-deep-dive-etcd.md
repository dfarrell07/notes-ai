---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, etcd, antithesis, robustness, fault-injection]
---

# Deep Dive: etcd Repo Automation

File-level deep dive of github.com/etcd-io/etcd. 20+ files read.
etcd has the most sophisticated fault injection and robustness
testing of any project surveyed.

## Key Findings for MCN

### 1. Antithesis Autonomous Fault Injection

Files: `.github/workflows/antithesis-test.yml`,
`tests/antithesis/`

Nightly 12-hour cron pushes 3 images to Antithesis's cloud
platform for autonomous exploration testing (random network
partitions, crashes, clock skew).

Three container images:

- **etcd-config**: Docker Compose config and K8s manifests
- **etcd-client**: traffic generation and validation binaries
- **etcd-server**: built with Antithesis instrumentation — converts
  all `// gofail` annotations to `assert.Reachable()` via sed,
  replaces internal verify functions with `assert.Always()`

Debugger workflow allows deterministic replay of specific fault
scenarios using `session_id`, `input_hash`, and `vtime`.

Verify workflow runs on every PR touching `tests/antithesis/` —
builds locally, runs Docker Compose 1-node and 3-node clusters.

**MCN**: Study this pattern for future chaos testing of MCN's
cross-cluster networking. The gofail-to-assertion conversion is
reusable for any Go project with failpoints.

### 2. Robustness Tests — Named Bug Reproduction

Files: `tests/robustness/main_test.go`,
`tests/robustness/scenarios/scenarios.go`,
`tests/robustness/failpoint/failpoint.go`

Two modes:

- **Exploratory**: randomized cluster size, timing, failpoints.
  36 failpoints including process kills, gofail panics, network
  blackhole/delay/drop, member replacement, downgrade/upgrade
- **Regression**: 10 named historical issues (Issue14370,
  Issue13766, etc.) each with exact cluster configs and failpoints

Weighted mixed-version distributions (60% current-only down to
10% quorum-last-version). LazyFS for filesystem fault injection.

Post-chaos validation: linearizability, watch correctness,
serializability with HTML visualization on failure.

The Makefile builds failpoint-enabled binaries from specific
historical releases for cross-version testing.

**MCN**: The named regression test pattern (tying specific bugs to
reproducible scenarios with exact version builds) is exceptional
engineering practice worth adopting for any distributed system.

### 3. TestGrid Flaky Test Auto-Triage

Files: `tools/testgrid-analysis/`, `scripts/measure-testgrid-flakiness.sh`

Daily cron fetches test results from TestGrid API (protobuf),
calculates 14-day rolling failure rates. Tests failing > 10%
(minimum 20 runs) are flagged.

For periodic tests: auto-creates GitHub issues with `type/flake`
and `help wanted` labels, structured markdown table of failures.
Deduplicates by checking existing open issues.

For presubmit tests: reports only, no issue creation.

**MCN**: Adoptable pattern even without TestGrid — query GitHub
Actions API for run history instead. The periodic-vs-presubmit
split is key for avoiding noise.

### 4. Dual-Location Release Asset Verification

File: `.github/workflows/verify-released-assets.yaml`

On every GitHub release publish:

1. Download all assets from GitHub release
2. Verify SHA256SUMS count matches file count
3. `sha256sum -c` on GitHub copies
4. Download same assets from Google Cloud Storage
5. Verify SHA256SUMS against GCS copies

~40 lines. Catches incomplete uploads or corrupted transfers.

**MCN**: Any project publishing to multiple locations should adopt
this. For MCN, verify Konflux-produced artifacts match registry
mirrors.

### 5. Cherry-Pick Bot Auto-Approval

Files: `.github/workflows/cherrypick-bot-ok-to-test.yaml`,
`.github/workflows/gh-workflow-approve.yaml`

Two-step process:

1. k8s-infra-cherrypick-robot (account ID 90416843) creates PR
   with `needs-ok-to-test` label
2. First workflow swaps label to `ok-to-test`
3. Second workflow finds all `action_required` workflow runs for
   that PR's HEAD SHA and approves them

Bot identification by hardcoded account ID for security. Two
separate workflows create clear audit trail.

**MCN**: Clean pattern for automating trusted-bot CI approval.
Eliminates manual approval bottleneck on release branches.

### 6. Cross-Module Dependency Consistency

From `scripts/test.sh`, `dep_pass` function:

Dumps all dependencies from all workspace modules, sorts them,
checks for any dependency appearing at different versions across
modules. Essential for multi-module monorepos.

**MCN**: If MCN becomes multi-module, adopt this check. Catches
version drift between modules.

### 7. golangci-lint — Domain-Specific Initialisms

File: `tools/.golangci.yaml`

15 linters. Notable:

- `revive` var-naming adds `GRPC` and `WAL` to strict initialisms
  (must be `GRPC` not `Grpc`, `WAL` not `Wal`)
- Disabled checks have explicit `TODO(fix)` comments — trackable
  tech debt
- `testifylint` requires f-assertions (`require.NoErrorf`) when
  messages are passed
- Formatters separate from linters (v2 format)

**MCN**: The domain-specific initialisms pattern is useful. Add
MCN's own terms (BGP, EVPN, VPN, VXLAN, etc.) to var-naming.
The TODO-commented disabled checks create actionable tech debt.

### 8. Branch-Specific Dependabot Cadences

File: `.github/dependabot.yml`

- GitHub Actions: weekly on all branches
- Go modules: weekly on main
- Docker: weekly on main, **monthly on release branches**

Separate `tools/mod` directory with its own Dependabot config
for tooling dependencies (prevents tool updates from polluting
main module).

**MCN**: Different cadences per branch type (weekly main, monthly
release) is practical. Separate tools module config is clean.

### 9. Release Process — DRY_RUN in CI

File: `scripts/release.sh`

Full release process:

1. Version validation + Go version check
2. Version bump in `api/version/version.go` + go.mod files
3. Tag creation (handles multi-module monorepo tags)
4. Binary builds for 4 Linux architectures
5. SHA256SUMS generation
6. Upload to Google Cloud Storage
7. Docker push to quay.io + gcr.io with multi-arch manifests
8. GitHub draft release creation
9. Version verification

The `release_tests_pass` function runs a full DRY_RUN release as
part of regular CI, including GPG key generation. This catches
release tooling regressions continuously.

**MCN**: DRY_RUN release testing in CI is excellent. Catches
release script bugs before actual release day.

### 10. LazyFS Filesystem Fault Injection

Files: Makefile (`install-lazyfs` target),
`tests/robustness/failpoint/failpoint.go`

Uses `dsrhaslab/lazyfs` v0.2.0 for filesystem-level fault
injection on single-node clusters. Installed as a first-class
Makefile target.

**MCN**: Interesting for testing data persistence guarantees.
Lower priority than network-level chaos for a networking operator.

## Top MCN Takeaways

1. **Named regression tests** — tie specific bugs to reproducible
   scenarios with exact version builds
2. **DRY_RUN release in CI** — catches release tooling regressions
3. **Dual-location asset verification** — ensure artifact integrity
4. **Cherry-pick bot auto-approval** — eliminate manual bottleneck
5. **Domain-specific initialisms** — enforce BGP, EVPN, VXLAN etc.
6. **Branch-specific update cadences** — weekly main, monthly
   release

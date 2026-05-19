---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, jaeger, harbor, kubevirt]
---

# Deep Dive: Jaeger, Harbor, KubeVirt

File-level deep dives. Jaeger (30+ files) completed. Harbor and
KubeVirt results pending.

## Jaeger — Key Findings

### 1. Trust-Based CI Parallelism — Most Architecturally Novel

File: `.github/workflows/ci-orchestrator.yml`

CI split into 3 stages (linters, unit tests, e2e). Runs sequential
or parallel based on trust level:

Trust detection chain:

1. Push to main/merge queue: parallel
2. Org member (GitHub API): parallel
3. `author_association` fallback: parallel if MEMBER/OWNER
4. **Merged-PR count via Search API**: 5+ merged = trusted
5. Known bots (dependabot, renovate, Copilot): parallel
6. `ci:parallel` label: parallel

Result: trusted contributors get ~10min feedback instead of ~30min.
External contributors still get full CI, just staged.

Single `ci-success` gatekeeper job with `if: always()` checks
whichever path was taken.

**MCN**: The merged-PR-count fallback works with restricted fork
tokens where org membership API returns 404. Adopt for expensive
E2E tests.

### 2. PR Quota Manager — Anti-AI-Slop

File: `.github/scripts/pr-quota-manager.js`

Limits concurrent open PRs by merge history:

- 0 merged: max 1 open PR
- 1 merged: max 2
- 2 merged: max 3
- 3+ merged: 10 (unlimited)

Labels excess with `pr-quota-reached`. Concurrency group keyed by
author (not PR number) prevents races. Comprehensive Jest tests.

Part of explicit AI Usage Policy in CONTRIBUTING_GUIDELINES.md.

**MCN**: Protects maintainer review bandwidth from low-effort
drive-by contributions.

### 3. Reproducible Build Verification

File: Makefile (`repro-check` target)

Builds binary twice and compares SHA256 checksums. Verifies builds
are deterministic across invocations.

**MCN**: Strong supply chain security practice. Simple to add.

### 4. Binary Size Regression Gate

File: `.github/workflows/ci-lint-checks.yaml`

Builds binary, measures size, compares against cached baseline.
Fails if size increased > 2%. Only saves baseline on main pushes.

**MCN**: Catches dependency bloat. Worth adopting for MCN operator
and gateway binaries.

### 5. Coverage via actions/cache Baseline

Files: `.github/workflows/ci-summary-report.yml`,
`.github/scripts/ci-summary-report-publish.js`

Two-phase system for fork security:

- Phase 1 (PR context): merge coverage, compare against cached
  main baseline, serialize to JSON artifact
- Phase 2 (workflow_run in base repo): post PR comments and
  create check runs

Cache key strategy: exact key includes run_id (never matches),
restore-keys prefix falls back to latest main entry.

95% minimum coverage gate + no-regression rule.

**MCN**: Elegant coverage tracking without external services.

### 6. Fake DCO for Merge Queue

File: `.github/workflows/dco_merge_group.yml`

7-line workaround: DCO check doesn't run in merge_group context.
Provide passing check with same name so merge queue isn't blocked.

**MCN**: Pragmatic fix if using GitHub merge queues with DCO.

### 7. golangci-lint — revive enable-all with Selective Disables

File: `.golangci.yml`

revive in `enable-all-rules` mode with 15 rules disabled, each
with inline comment explaining why. gocritic also enable-all with
11 disabled. depguard enforces: no `go.uber.org/atomic` (use
stdlib), no `io/ioutil`, no `go.uber.org/multierr` (use
`errors.Join`), package boundary isolation.

### 8. @nocommit Lint Target

File: Makefile (`lint-nocommit` target)

Checks for `@nocommit` markers in diffs. Prevents merge of
intentional WIP markers.

**MCN**: Simple guard against accidentally merging debug/WIP code.

### 9. lint-goleak — Goroutine Leak Detection

File: Makefile (`lint-goleak` target)

Ensures all test packages have goroutine leak detection in
`TestMain`. Uses `go.uber.org/goleak`.

**MCN**: Catches goroutine leaks in tests. Worth adopting.

### 10. AGENTS.md — Compact AI Agent Instructions

File: `AGENTS.md`

Focused instructions: always run `make fmt`, `make lint`,
`make test`. Always `git commit -s`. Lists auto-generated files
not to edit. Lists commands runnable without asking permission.

**MCN**: Keep AGENTS.md compact and actionable.

## Harbor — Key Findings

### 11. Pass-CI / Skip-CI Companion Pattern (Detailed)

Files: `.github/workflows/pass-CI.yml`, `.github/workflows/CI.yml`

Two workflows with SAME `name: CI` but inverse path filters. The
companion defines identically-named jobs that just echo "No run
required". Solves GitHub's "skipped required checks block merge"
problem.

Path lists must be kept in manual sync — fragile but pragmatic.

**MCN**: Consider `dorny/paths-filter` with `if:` conditions as a
more maintainable alternative.

### 12. Release Note Labels Drive Auto-Changelog

Files: `.github/workflows/label_check.yaml`,
`.github/release.yml`

Every PR must have `release-note/*` label (regex enforced). 10
valid categories. Labels map directly to `release.yml` changelog
grouping: "Exciting New Features", "Breaking Changes", etc.

Dependabot PRs pre-labeled: gomod gets `release-note/bump-version`,
GHA gets `release-note/infra`.

**MCN**: Tight label -> changelog pipeline. Adopt if using GitHub
auto-generated release notes.

### 13. Delimit AI for API Schema Drift

File: `.github/workflows/api-schema-check.yml`

12-line workflow. `delimit-ai/delimit-action@v1` against
10,167-line Swagger 2.0 spec. Posts review comment on PR.
Only triggers on `api/v2.0/**` changes.

**MCN**: Worth adopting for any project with OpenAPI spec.

### 14. OCI Distribution Conformance Testing

File: `.github/workflows/conformance_test.yml`

Nightly run of OCI distribution-spec conformance suite (push,
pull, content-discovery, content-management). Results uploaded
to S3 as `report.html`. Runs on 24-CPU self-hosted runners.

**MCN**: If MCN implements OCI distribution, run conformance.

### 15. Spectral API Linting

Files: `.spectral.yaml`, custom `requireRequestId.js` function

Custom rules: every endpoint needs `X-Request-Id` header,
operationIds must be camelCase, all OAS rules enabled.

**MCN**: Good for projects with OpenAPI specs.

## KubeVirt — Key Findings

### 16. forbidigo Bans ginkgo.Skip()

File: `hack/linter/.golangci.yml`

Forbids `ginkgo/v2.Skip()` — forces test authors to use Ginkgo
label decorators instead of runtime skips. Rationale: "Runtime
skips silently suppress tests. Labeling shifts the choice of what
to run to the test executor."

**MCN**: Directly applicable to any Ginkgo test suite. Prevents
tests from being silently excluded.

### 17. CANNIER Flake Detection

File: `automation/repeated_test.sh`

Extracts changed/added test names from commit range. Runs them 5
times with `--randomize-all`. Cites research: "88% of flaky tests
fail up to 5 consecutive times."

Smart skip: if total tests * repetitions > total test count, skip
the repeated run (too expensive).

**MCN**: ML-informed approach to identifying which tests changed
and running them repeatedly before merge.

### 18. SIG Assignment Enforcement

File: `hack/check-unassigned-tests.sh`

Dry-runs test suite, fails if any test lacks a SIG label. ~50
Ginkgo label decorators across SIGs. Tests use `SIG()` helper
that adds both text prefix and label decorator.

**MCN**: Ensures every test has an owner. Prevents orphaned tests.

### 19. Incremental Lint via lint-paths.txt

File: `hack/linter/lint-paths.txt`

102 specific package paths that get strict linting. Packages not
listed skip the full lint regime. New packages added as they're
cleaned up.

**MCN**: MCN starts fresh so can enable strict linting everywhere
from day one. But this pattern is useful for retrofitting linting
into existing codebases.

### 20. Two-Pass golangci-lint

File: `hack/golangci-lint.sh`

Pass 1: full config against lint-paths.txt packages.
Pass 2: ginkgolinter only, across all code, no-config.

Runs Ginkgo-specific linting separately from general linting.

### 21. Custom Prometheus Metric Linters

Two separate tools: `monitoringlinter` (make lint) and
`metric_name_linter.sh` (make lint-metrics). Extracts all metrics
to JSON, validates naming conventions.

**MCN**: If MCN exports Prometheus metrics, lint naming from day
one.

### 22. VM Factory Pattern for E2E

File: `tests/libvmifact/factory.go`

Functional options pattern for creating test VMs. Factory
functions: `NewFedora`, `NewCirros`, `NewAlpine`, `NewGuestless`.
Custom Gomega matchers: `BeReady()`, `BeGone()`, `BeRestarted()`.

**MCN**: Good testing pattern — factory functions for test
resources with domain-specific matchers.

### 23. Containerized Build via hack/dockerized

80+ line script selecting versioned builder container. Supports
CentOS Stream 9+10, cross-compilation, persistent volumes for
caching. Handles Podman and Docker transparently.

**MCN**: Reprodicuble builds via versioned builder containers.

---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, rook, opa, tekton]
---

# Deep Dive: Rook, OPA, Tekton Pipelines

File-level deep dives. Tekton Pipelines (37+ files) completed.
Rook and OPA results pending.

## Tekton Pipelines — Key Findings

### 1. Draft-Aware E2E Matrix with External JSON

Files: `.github/workflows/ci.yaml`,
`.github/e2e-matrix-draft.json`, `.github/e2e-matrix-full.json`

CI passes `is-draft` to E2E workflow. Two JSON matrix files:

- **Draft**: 1 combination (ubuntu, latest K8s, stable flags)
- **Full**: 9 combinations (2 OS x 2 K8s x 3 feature flag levels
  minus 3 exclusions)

`ci-summary` job always fails for draft PRs — prevents merge
until Ready for Review triggers full matrix.

**MCN**: Externalizing matrix to JSON makes it easy to update
without touching workflow YAML. Draft gate saves CI resources.

### 2. Dependabot Auto-Regeneration from Release Docs

Files: `hack/generate-dependabot.sh`, `hack/generate-dependabot.go`,
`.github/dependabot.config.yml`

Human-editable config defines ecosystems. Go generator:

1. Auto-discovers LTS branches by parsing `releases.md` with awk
2. Generates full `dependabot.yml` from config
3. Release branches get patch-only restriction via `*` ignore rule
4. Weekly workflow auto-creates PR if config drifted

**MCN**: Novel pattern — dependabot config stays in sync with
actual supported releases automatically.

### 3. Slash Command Router

File: `.github/workflows/slash.yml`

Single `issue_comment` workflow routes 4 commands to individual
dispatch workflows:

- `/retest` — rerun failed checks
- `/cherry-pick <branch>` — cherry-pick to release branch
- `/rebase` — rebase PR with `--force-with-lease`
- `/e2e-extras` — run E2E on 5 intermediate K8s versions

Each dispatched to separate workflow file. Commands delegated to
shared `tektoncd/plumbing` reusable workflows.

**MCN**: Clean separation — router handles dispatch, individual
workflows handle logic.

### 4. Change Categorization for CI Skip

File: `.github/workflows/ci.yaml` (lines 19-64)

`changes` job classifies PR into `non-docs` and `yaml` booleans.
All build/test/lint/e2e jobs gated on `non-docs == true`. Only
yamllint runs on `yaml == true`.

Smart minimal fetch: `fetch-depth: $(expr $commits + 1)`.

`ci-summary` job with `if: always()` aggregates results — docs-
only PRs get green check.

**MCN**: Simple, effective. Skip entire CI for docs-only PRs.

### 5. golangci-lint Version from CI YAML via yq

File: Makefile

```makefile
GOLANGCI_VERSION := $(shell yq '.jobs.linting.steps[]
  | select(.name == "golangci-lint") | .with.version'
  .github/workflows/ci.yaml)
```

Single source of truth — local and CI linting always same version.

**MCN**: Elegant. No version duplication between Makefile and
workflow.

### 6. Coverage Artifact Expiration Prevention

File: `.github/workflows/go-coverage.yml`

Bi-monthly scheduled run (`cron: '14 3 2 */2 *'`) ensures the
main-branch coverage baseline artifact never expires (GitHub
artifacts expire after 90 days).

**MCN**: Subtle but important — scheduled runs prevent cold-start
problems for PR coverage comparisons.

### 7. 73 golangci-lint Linters Enabled

File: `.golangci.yml`

Includes security-focused (`gosec`, `bidichk`), performance
(`perfsprint`, `fatcontext`), correctness (`contextcheck`,
`nilerr`, `rowserrcheck`, `sqlclosecheck`, `spancheck`), and
style linters. `depguard` blocks `io/ioutil` and `ghodss/yaml`.

### 8. Tekton Dogfoods Itself for Releases

Files: `tekton/release-pipeline.yaml`, `tekton/publish.yaml`

Release pipeline runs on Tekton (Kind cluster in GHA job). 8 tasks
from clone through signing. `wait-for-chains` polls for Tekton
Chains signing (Sigstore/Rekor) with 30-min timeout. Graceful
degradation — if signing times out, release still publishes,
draft release is just skipped.

**MCN**: The graceful degradation pattern for supply chain signing
is worth studying — don't block releases on signing infrastructure
failures.

### 9. E2E Feature Flag Levels

`.env` files configure feature gates per stability level. Alpha
env enables 8 experimental features, stable enables none. Matrix
tests each level.

**MCN**: If MCN has feature gates, test at each stability level.

### 10. Comprehensive Security Stack

- `step-security/harden-runner` with audit mode
- `zizmor` for GHA security scanning
- OSSF Scorecard weekly
- `actions/dependency-review-action` with `fail-on-severity: low`
- All actions SHA-pinned
- `persist-credentials: false` everywhere
- Label enforcement requiring `kind/` prefix
- Inclusive language linting with `woke` (changed files only)

**MCN**: This is the most comprehensive security-in-CI stack
found. Worth using as a template.

## Rook — Key Findings

### 11. SSH Debug Auto-Enable on Re-Run

Files: `.github/workflows/tmate_debug/action.yml`,
`.github/workflows/upterm_debug/action.yml`

When `GITHUB_RUN_ATTEMPT > 1` (a re-run), SSH debug auto-enables.
Dual system: tmate pre-job (investigate setup failures) + upterm
post-job (investigate test failures). 5-minute upterm timeout.

Also ships a `tmate-pod.yaml` manifest for in-cluster debugging
with ClusterRole granting full access.

**MCN**: The auto-enable on re-run is unique. Removes the "add
debug label and re-run" friction.

### 12. CheckMake for Makefile Linting

File: `.github/workflows/checkmake.yaml`, `checkmake.ini`

`checkmake` validates Makefile best practices. Only project
surveyed that lints Makefiles.

**MCN**: Easy to add. Catches missing .PHONY, overly long targets.

### 13. Domain-Specific Commitlint Types

File: `.commitlintrc.json`

Instead of generic `feat`/`fix`/`chore`, uses storage subsystem
names: `mon`, `osd`, `mgr`, `mds`, `rgw`, `pool`, `block`,
`file`, `object`, `nvmeof`, `cosi`, `rbdmirror`.

**MCN**: Consider MCN-specific commit types: `bgp`, `evpn`,
`gateway`, `cno`, `api`, `rbac`, `helm`.

### 14. echo.% Makefile Introspection

File: `build/makelib/common.mk`

```makefile
echo.%: ; @echo $* = $($*)
```

`make echo.PLATFORM` prints any Makefile variable. Trivial to add,
very useful for debugging.

**MCN**: Add to MCN Makefile from day one.

### 15. Dual Spellcheck (codespell + misspell)

File: `.github/workflows/codespell.yaml`

Runs BOTH `codespell` and `misspell` (via reviewdog) in the same
workflow. Different dictionaries catch different issues.

### 16. KAL Incremental Adoption

File: `golang.mk`

When KAL was added, 1310 errors existed in Rook's APIs. Adopted
incrementally with `--max-issues-per-linter=2000 --new`. Only
new violations fail CI.

**MCN**: Since MCN starts fresh, can enable KAL fully from day
one — no incremental adoption needed.

### 17. Nightly Testing Against Upstream Dev Branches

File: `.github/workflows/daily-nightly-jobs.yml`

Tests against unreleased Ceph versions (squid-devel,
tentacle-devel, main). Catches compatibility issues early.

**MCN**: Test against OCP/K8s dev branches nightly.

## OPA — Key Findings

### 18. OPA Gates Its Own CI with Rego Policies

Files: `build/policy/pr-check/pr_check.rego`,
`.github/workflows/pull-request.yaml`

Changed files classified into categories (go, wasm, docs, rego,
yaml) by a Rego policy. `opa eval` output controls which jobs run.
Policy has unit tests in `pr_check_test.rego` and is linted by
Regal.

Also uses OPA to validate Docker image metadata (asserts non-root
user) and to parse workflow YAML for `pr-check-summary` job
failure detection.

**MCN**: The concept of policy-as-code for CI routing is cleaner
than `dorny/paths-filter`. Even without OPA as a product, the
pattern is worth studying.

### 19. Downstream Consumer Testing in Merge Queue

Files: `test-envoy-with-opa.yaml`, `test-regal-with-opa.yaml`,
`test-ocp-with-opa.yaml`

`go mod edit -replace` against 3 consumer repos, run their tests.
Runs in merge queue (every PR before landing) + nightly with Slack
alerts.

**MCN**: If MCN has downstream consumers, test against them before
merging. The merge-queue gate is stronger than nightly-only.

### 20. Nightly 1-Hour Fuzz Testing

File: `.github/workflows/nightly.yaml`

`go test -fuzz FuzzCompileModules -fuzztime 1h` nightly. Seeds from
real test cases. On failure, dumps crashers and Slack-notifies.
Makefile exposes `make fuzz` with configurable `FUZZ_TIME`.

**MCN**: 1-hour budget is practical sweet spot for nightly fuzzing.

### 21. govulncheck Against Latest Release Tag

File: `release-vulnerability-check.yaml`

Runs govulncheck nightly against the LATEST RELEASE TAG, not just
main. Catches vulnerabilities in shipped code.

**MCN**: Dual approach (main + latest release) catches CVEs in
code users actually run.

### 22. Benchmark Regression with 25% Threshold

File: `gobenchdata-checks.yml`

Automated benchmark comparison with 25% tolerance. Posts PR
comments on regressions. Uses `gobenchdata` for storage and
comparison.

**MCN**: Automated performance regression detection with
configurable threshold.

### 23. testscript/txtar for CLI Smoke Tests

5 txtar scripts testing `opa version`, `opa eval`, `opa exec`,
`opa inspect`, `opa build`. Declarative, version-controllable.

**MCN**: If MCN has a CLI, txtar smoke tests are cleaner than
bash scripts.

### 24. 4 Docker Image Variants per Release

Standard, debug, static, static-debug. Uses Chainguard base images
(`glibc-dynamic`, `static`, `busybox`). Cross-compilation via Zig.

### 25. AGENTS.md — Anti-AI-Spam Policy

"The most important rule when working on this project is not to
post comments on issues or PRs which are AI-generated."

**MCN**: Consider what AI contribution policy MCN should have.

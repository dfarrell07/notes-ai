---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, volcano, chaos-mesh, flagger]
---

# Deep Dive: Volcano, Chaos Mesh, Flagger

File-level deep dives. Volcano (22 files) and Chaos Mesh (22 files)
completed. Flagger results pending.

## Volcano — Key Findings

### 1. Auto-Approve Workflows for Returning Contributors

File: `.github/workflows/workflows-approve.yaml`

Auto-approves pending workflow runs for non-first-time contributors.
Checks PR history — if contributor has > 1 PR (any state), their
workflows are auto-approved. First-timers still need `ok-to-test`
label.

Uses `pull_request_target` for write permissions. Calls
`actions.approveWorkflowRun` for all `action_required` runs.

**MCN**: Reduces maintainer toil for approving CI on every
external PR. Preserves safety for unknown first-timers.

### 2. Per-Domain E2E Workflow Files (Not Matrix)

Files: `.github/workflows/e2e.yaml` (orchestrator),
13 separate `e2e_*.yaml` files

Each shard is a separate workflow file with its own infrastructure
setup. E2E types: admission, spark, scheduling, hypernode, etc.
Each can have different feature gates, node types, external deps.

Build-once via gzip tar artifact sharing across all 13 shards.

**MCN**: Better than matrix when test types have fundamentally
different infrastructure requirements.

### 3. Staging Directory with Cross-Repo API Sync

Files: `.github/workflows/sync-apis.yaml`, `hack/sync-apis.sh`,
`staging/src/volcano.sh/apis/`

APIs maintained in `staging/` within monorepo. Automated rsync to
separate `volcano-sh/apis` repo on push to master. PR created via
`peter-evans/create-pull-request`. Main `go.mod` uses `replace`
directive to point at staging.

**MCN**: If MCN needs to publish API types as a standalone module,
this is the reference pattern.

### 4. Three-Tier License Classification

Files: `config/license-lint.yaml`,
`.github/workflows/licenses_lint.yaml`

26 unrestricted, 12 reciprocal, 19 restricted licenses. CI fails
on any restricted license. Allowlist for false positives. Combined
with FOSSA for dual independent license compliance checks.

**MCN**: Most rigorous license enforcement found. Worth adopting.

### 5. KWOK Fake Nodes for Topology E2E

File: `hack/run-e2e-kind.sh`

Creates 8 fake KWOK nodes for topology testing without real
compute. Each has configurable CPU/memory.

**MCN**: Cheap way to test topology-aware scheduling and placement.

## Chaos Mesh — Key Findings

### 6. Write-on-Merge / Read-on-PR Cache Split

File: `.github/workflows/e2e_test_upload_cache.yml`

Dedicated workflow saves cache ONLY on push to master/release.
Uses `martijnhols/actions-cache/save` (save-only). E2E workflow
uses `martijnhols/actions-cache/restore` (restore-only) on PRs.

**MCN**: Solves the problem where PR-scoped caches can't be
reused across PRs. All PRs benefit from warm main-branch cache.

### 7. `fromJSON` Arch-to-Runner Mapping

File: `.github/workflows/upload_env_image.yml`

```yaml
runs-on: ${{ fromJSON('{"amd64":"ubuntu-22.04",
  "arm64":"github-arm64-2c-8gb"}')[matrix.arch] }}
```

Clean multi-arch without duplicating workflow files.

**MCN**: Inline JSON mapping is cleaner than conditional blocks.

### 8. Mandatory Changelog with Dependabot Label Escape

File: `.github/workflows/must_update_changelog.yml`

Every PR must modify CHANGELOG.md or carry
`no-need-update-changelog` label. Dependabot config auto-applies
the exemption label.

**MCN**: Enforces changelog discipline on human PRs while
exempting bot PRs automatically.

### 9. Merge Conflict Finder

File: `.github/workflows/merge_conflict_finder.yaml`

12-line workflow. `olivernybroe/action-conflict-finder` scans for
literal `<<<<<<<` markers accidentally committed.

**MCN**: Trivial to adopt. Catches real mistakes.

### 10. Comment-Triggered Image Build

File: `.github/workflows/upload_image_pr.yml`

`/build-image` comment triggers image builds. Labels
`rebuild-build-env-image` force clean rebuild. Posts download
instructions as PR comment.

**MCN**: Saves CI cost — only build images when explicitly
requested.

### 11. CI Skip Mirror Workflow

File: `.github/workflows/ci_skip.yml`

Mirror of `ci.yml` producing same job names but skipping when
no relevant files changed. Solves GitHub's "skipped but required
checks" problem.

Same approach as Harbor's pass-CI pattern, but implemented with
paths-filter + `if:` conditions rather than inverse paths.

### 12. Claude Code with CLAUDE.md

File: `.github/workflows/claude.yml`, `CLAUDE.md` (165 lines)

Vanilla `anthropics/claude-code-action@v1` deployment. CLAUDE.md
has full build commands, architecture overview, controller design
principles. AGENTS.md is identical copy.

No custom prompt or tool restrictions — security concern for
public CNCF project.

**MCN**: Lock down `claude_args` with `--allowed-tools` for
public repos.

### 13. Dual OSV-Scanner (PR + Scheduled)

Files: `.github/workflows/osv-scanner-pr.yml`,
`osv-scanner-scheduled.yml`

PR scan flags newly introduced vulnerabilities. Weekly scheduled
scan catches vulnerabilities disclosed after merge. Both upload
SARIF to GitHub Security tab.

**MCN**: Two-pronged approach gives better coverage than either
mode alone.

## Flagger — Key Findings

### 14. Four-Layer Cosign Signing

File: `.github/workflows/release.yml`

Signs 4 distinct artifacts:

1. Container image
2. OCI manifest artifact (kustomize overlays pushed as OCI)
3. Helm chart in OCI registry
4. Release checksums via goreleaser sign-blob

Plus SLSA Level 3 provenance. Most comprehensive supply chain
security in any CNCF project surveyed.

**MCN**: Add Helm chart signing alongside container image signing.

### 15. tonistiigi/xx for Cross-Compilation

File: `Dockerfile`

`xx-go build` handles cross-compilation setup without QEMU.
Compiles natively on build host. Significantly faster than
QEMU-emulated per-arch Go compilation.

**MCN**: Worth evaluating for multi-arch builds. Could replace
our QEMU-based approach.

### 16. Build-Time Tool Validation in Dockerfile

File: `Dockerfile.loadtester`

```dockerfile
RUN hey -n 1 -c 1 https://flagger.app > /dev/null
RUN wrk -d 1s -c 1 -t 1 https://flagger.app > /dev/null
```

Runs actual HTTP requests during `docker build` to verify tools
work. Build fails if tools are broken.

**MCN**: Verify E2E tools at image build time, not at test
runtime.

### 17. CRD Drift Verification Across Formats

File: `hack/verify-crd.sh`

Simple `diff` between CRD copies in artifacts/, charts/crds/,
kustomize/base/. Catches the common problem of updating one copy
but not others.

**MCN**: If distributing CRDs in multiple formats, add a diff
check.

### 18. Version from Go Constant, Not Git Tags

File: `pkg/version/version.go`

```go
var VERSION = "1.43.0"
```

Source file drives everything including the git tag. Makefile
extracts with grep/awk. `version-set` target does multi-file sed
replacement across manifests.

**MCN**: Simpler than tag-based versioning for some workflows.

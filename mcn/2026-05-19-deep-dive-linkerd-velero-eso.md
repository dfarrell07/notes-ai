---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, linkerd, velero, external-secrets]
---

# Deep Dive: Linkerd, Velero, External Secrets Operator

File-level deep dives. Linkerd (24+ files) and Velero (22+ files)
completed. External Secrets results pending.

## Linkerd — Key Findings

### 1. Cross-Repo Version Sync via .proxy-version

Files: `.proxy-version`, `.github/workflows/sync-proxy.yml`

Single-line file tracking proxy version. `workflow_dispatch`
workflow checks latest release from `linkerd/linkerd2-proxy`,
creates branch `bot/sync-proxy/$VERSION`, opens PR with release
notes link. SHA256 verification on download.

**MCN**: Clean cross-repo version tracking pattern. A
`.component-version` file per dependency with auto-PR workflow
improves traceability vs manual bumps.

### 2. Two-Tier Flaky Test Retry

Files: `.github/workflows/rerun.yml`, integration.yml, go.yml

Generic `rerun.yml` invokable from any workflow:

```yaml
integrations-retry:
  if: failure() && fromJSON(github.run_attempt) < 3
       && needs.build-ok.result == 'success'
```

Key design:

- `fromJSON(github.run_attempt) < 3` caps at 3 attempts
- Only retries if builds passed (test flakes, not build failures)
- `gh run rerun --failed` re-runs only failed jobs
- Rust tests use nextest's built-in per-test retry

**MCN**: Most sophisticated retry strategy found. The build-success
guard prevents infinite retry of broken builds.

### 3. CI-Enforced Version Consistency

Files: `.github/workflows/actions.yml`, justfile

Three checks:

1. actionlint on all workflow files
2. `check-action-images` verifies workflow container images match
   devcontainer
3. Rust toolchain version in `rust-toolchain.toml` cross-checked
   against Docker image tags in workflow files

**MCN**: Prevents devcontainer/CI version drift. No other surveyed
project enforces this.

### 4. Justfile Instead of Makefile (582 lines)

Linkerd uses `just` instead of `make`. Notable patterns:

- Random bogus registry for test images: `"test.l5d.io/" + random`
  — impossible to accidentally pull wrong images
- CI-aware formatting with `cargo-action-fmt`
- Internal recipes prefixed with `_` for composability
- Smart tool detection (cargo-nextest vs cargo test)

**MCN**: The random bogus registry is clever safety. Stay with
Makefile but study the composability patterns.

### 5. Wolfi/apko for Base Images

File: proxy-runtime.yml (apko manifest)

Defines proxy runtime base image using apko (Wolfi/Chainguard),
not a Dockerfile. Produces minimal, reproducible, distroless-like
images from a YAML manifest.

**MCN**: Worth evaluating for reproducible base images. More
deterministic than Dockerfile + package manager.

### 6. Centralized Dev Repo

Every workflow references `linkerd/dev/actions/setup-tools@v48`.
Dev container, justfile tool, lint commands all from one versioned
repo. Developers and CI use the same `ghcr.io/linkerd/dev:v49`
image.

**MCN**: Strong pattern if MCN ever spans multiple repos.

## Velero — Key Findings

### 7. Go Module Filepath Unicode Validation

File: `.github/workflows/pr-filepath-check.yml`

Inline Python script validates every filename character against
Go's module zip format (`fileNameOK()`). Added after real incident
(PR #9552) where Unicode LEFT-TO-RIGHT MARK (U+200E) broke builds.

Provides actionable `git mv` fix commands and GitHub `::error`
annotations for inline PR feedback.

**MCN**: Unique check — catches invisible characters that break
Go builds. Self-contained, zero dependencies, directly portable.

### 8. Prow-as-GitHub-Actions

File: `.github/workflows/prow-action.yml`

`jpmcb/prow-github-actions@v1.1.3` provides Prow slash commands
(`/approve`, `/area`, `/assign`, `/cc`, `/close`, `/hold`, `/kind`,
`/milestone`, `/retitle`, etc.) without Prow infrastructure.

`/lgtm` deliberately excluded — want changelog enforcement first.

**MCN**: Lightweight Prow UX without Prow infrastructure. Good
for projects that want familiar commands in pure GHA.

### 9. Two-Layer Spell Checking

File: `.github/workflows/pr-codespell.yml`

Layer 1: codespell for generic spelling.
Layer 2: project-specific terminology:

- lowercase "kubernetes" must be "Kubernetes"
- "on-premise" must be "on-premises"
- "back-up" must be "backup"
- "whitelist"/"blacklist" — inclusive language

Files opt out with `Velero.io word list : ignore` comment.

**MCN**: Practical terminology enforcement. The opt-out mechanism
handles edge cases.

### 10. Per-PR Changelog Files with make new-changelog

Files: `hack/changelog-check.sh`, `hack/release-tools/changelog.sh`

Every PR must include `changelogs/unreleased/{PR_NUMBER}-{LOGIN}`.
`make new-changelog` auto-generates from PR metadata via `gh` CLI.
Label exemptions: `kind/changelog-not-required`, `Design`,
`Website`, `Documentation`.

At release: aggregate all files into formatted markdown, then
`git rm changelogs/unreleased/*`.

**MCN**: Similar to Contour's pattern but simpler (no category in
filename). The `make new-changelog` auto-fill is convenient.

### 11. GoReleaser Dry-Run on Config Changes

File: `.github/workflows/pr-goreleaser.yml`

When `.goreleaser.yml` or `hack/release-tools/goreleaser.sh` is
modified, runs full `goreleaser release --snapshot` dry-run.
Catches config bugs before they block a real release.

**MCN**: Essential if using GoReleaser. Also validates checksums
and cross-compilation.

### 12. Dynamic K8s Version Matrix from Docker Hub

File: `.github/workflows/e2e-test-kind.yaml`

Scrapes `kindest/node` tags from Docker Hub at runtime to
generate the test matrix. Never goes stale, no manual updates.

**MCN**: Elegant. Matrix auto-discovers available K8s versions.

### 13. nolintlint Requiring Explanation + Specific Linter

File: `.golangci.yaml`

Every `//nolint` must specify the linter AND include an
explanation. No bare `//nolint` allowed.

**MCN**: Good governance for lint suppression.

## Top MCN Takeaways (Combined)

1. **Two-tier flaky retry** (Linkerd) — reusable workflow with
   attempt cap and build-success guard
2. **Filepath Unicode validation** (Velero) — catches invisible
   build-breaking characters
3. **Cross-repo version sync** (Linkerd) — version file + auto-PR
4. **CI version consistency checks** (Linkerd) — devcontainer ==
   CI == toolchain
5. **Dynamic K8s version matrix** (Velero) — auto-discover from
   Docker Hub
6. **Random bogus registry** (Linkerd) — prevent accidental wrong
   image pulls in tests
7. **GoReleaser dry-run on config changes** (Velero)
8. **Two-layer spelling** (Velero) — generic + project terminology

## External Secrets Operator — Key Findings

### 14. /lgtm with CODEOWNERS-Aware Role Checking

Files: `.github/workflows/lgtm.yml`,
`.github/scripts/lgtm-processor.js` (255 lines with unit tests)

Prow-like `/lgtm` without Prow. The processor reads
`CODEOWNERS.md` (markdown format), checks commenter's team
membership against required reviewer roles for changed files.

- Maintainer (matches `*`) — LGTM applied immediately
- Provider team member — LGTM applied, reports uncovered roles
- No matching role — rejection with required roles list

Auto-removes `lgtm` label on `synchronize` (new commits pushed).
Unit tested via `lgtm-processor-test.js`.

30+ provider-specific reviewer teams.

**MCN**: Most granular LGTM implementation found. The auto-remove
on push prevents stale approvals. Worth studying if MCN has
sub-component ownership.

### 15. Comprehensive harden-runner Adoption

20 of 24 workflows use `step-security/harden-runner` as first step
of every job. All with `egress-policy: audit` (observe, not block).
Consistent SHA pinning with `@sha # vX.Y.Z` comment pattern.

**MCN**: ESO proves harden-runner scales to 20+ workflows without
friction. Start with audit mode.

### 16. CodeQL with Expanded Threat Model

File: `.github/config/codeql-config.yaml`

Uses `threat-models: local`, disables default queries, pulls from
4 query packs including Trail of Bits and GitHub Security Lab
community packs. Far more aggressive than default CodeQL.

**MCN**: The community query packs catch issues default CodeQL
misses. Low effort to add.

### 17. SBOM Deduplication for Rekor Size Limits

File: `.github/actions/sign/action.yml` (210 lines)

Dual SBOMs: image SBOM (OS + libs) and Go modules SBOM (source
dependencies). Custom `hack/dedupe-spdx-gomod.sh` keeps SBOMs
under 10MB (Rekor limit). Falls back to dropping file ownership
data if still too large. Verifies all attestations after
attachment.

**MCN**: Only project that handles SBOM size limits gracefully.

### 18. check-diff Gate for Generated Code

File: Makefile (`reviewable` and `check-diff` targets)

`reviewable` runs ALL generation: codegen, docs, manifests, helm,
schema, lint, license check, tests, `go mod tidy` on 6+ modules.
`check-diff` runs `reviewable` then asserts clean git state.

**MCN**: Comprehensive "nothing drifted" CI gate. The most
thorough generated-code verification pattern found.

### 19. Skip Duplicate Actions

`fkirc/skip-duplicate-actions` prevents redundant CI runs on same
commit (e.g., push + PR open triggers). Used in CI and zizmor.

**MCN**: Simple optimization — avoids double CI runs.

### 20. CRD Backward Compatibility Testing

Uses `cty` (crd-to-sample-yaml) to test CRDs for breaking changes
via `make test.crds`.

**MCN**: Another CRD compat tool alongside crdify and go-apidiff.

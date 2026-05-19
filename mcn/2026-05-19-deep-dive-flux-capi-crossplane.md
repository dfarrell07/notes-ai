---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, flux, cluster-api, crossplane]
---

# Deep Dive: Flux, Cluster API, Crossplane

File-level deep dives. Flux completed (20+ files). Cluster API and
Crossplane results pending — will append when available.

## Flux (fluxcd/flux2) — Key Findings

### 1. AGENTS.md — Best AI Agent Guidance Found

File: `AGENTS.md` (152 lines)

The most comprehensive AI-agent instruction file in any surveyed
project. Covers:

- Contribution workflow (DCO handling, commit format)
- Code quality rules (no secrets in logs, no path traversal, no
  command injection, no panics)
- Project architecture overview
- CLI conventions and testing patterns
- Key quote: "Do not add Signed-off-by or Co-authored-by trailers
  with your agent name. Only a human can legally certify the DCO."

**MCN**: Create an AGENTS.md alongside CLAUDE.md. AGENTS.md is
tool-agnostic (works with any AI agent), CLAUDE.md is Claude-specific.

### 2. Per-Registry SLSA Provenance

File: `.github/workflows/release.yaml`

Three separate SLSA Level 3 provenance attestations:

1. Binary checksums (via `generator_generic_slsa3`)
2. Docker Hub image (via `generator_container_slsa3`)
3. GHCR image (via `generator_container_slsa3`)

Each registry gets its own provenance with distinct credentials.
Most thorough SLSA implementation found across all projects.

**MCN**: If publishing to multiple registries, generate separate
provenance per registry.

### 3. CRD JSON Schema at Release Time

File: `.github/workflows/release.yaml`

```yaml
- name: Build CRDs
  run: kustomize build manifests/crds > all-crds.yaml
- name: Generate OpenAPI JSON schemas from CRDs
  uses: fluxcd/pkg/actions/crdjsonschema@...
```

Schemas bundled as `crd-schemas.tar.gz` release asset. Enables
IDE validation of custom resources.

**MCN**: Adopt for API documentation and developer experience.

### 4. Multi-Distribution Testing (Kind + K3s + OpenShift)

File: `.github/workflows/conformance.yaml`

Three distributions tested:

- Kind on ARM64 with Calico CNI (not KindNet — enables
  NetworkPolicy testing)
- K3s via `replicatedhq/replicated-actions/create-cluster`
  with 20-min TTL
- OpenShift 4.20-okd via same Replicated provisioning

OpenShift-specific manifests handle SCC (Security Context
Constraints) via dedicated `manifests/openshift/` kustomize
overlay.

**MCN**: The Replicated cluster provisioning with built-in TTL
cleanup is worth evaluating for real multi-distro testing.

### 5. Component Auto-Update Workflow

File: `.github/workflows/update.yaml`

On push to main, queries GitHub releases API for all 7 controller
repos. Bumps versions across manifests AND go.mod. Creates PR with
changelog links. Skips release candidates.

**MCN**: Adaptable for any multi-component operator.

### 6. Manifest Embedding Pipeline

Five-step pipeline: Kustomize bases -> bundle.sh -> generated YAML
-> `go:embed` -> CLI binary. The CLI binary contains all controller
manifests at compile time.

Sentinel file pattern (`cmd/flux/.manifests.done`) with `rwildcard`
dependency tracking in Makefile — rebuilds if any YAML changes.

**MCN**: If MCN has a CLI that deploys the operator, this embedding
pattern is the reference.

### 7. Declarative Label Sync

File: `.github/labels.yaml`

15 labels defined in YAML, synced to GitHub on push to main via
shared workflow. Eliminates label drift.

**MCN**: Simple, useful. Define labels in code, sync automatically.

### 8. No golangci-lint

Flux uses no golangci-lint at all. Relies on `go fmt`, `go vet`,
and CodeQL via shared workflows. This is unusual but deliberate —
they prefer standard Go tooling.

**MCN**: Don't follow this pattern. golangci-lint catches
significantly more issues than standard tooling alone.

### 9. Cosign Signing — Everything

GoReleaser signs checksums with cosign (covers all binaries).
Docker images and manifest lists all signed. OCI manifests signed
on both Docker Hub and GHCR. All keyless via OIDC.

### 10. AUR Publishing

Three Arch Linux AUR packages auto-published via GoReleaser
post-hooks. Handles pkgrel incrementing for same-version
re-releases.

**MCN**: Niche. Only relevant if targeting Arch Linux users.

## Cluster API (kubernetes-sigs/cluster-api) — Key Findings

### 11. Changelog-as-Release-Trigger

File: `.github/workflows/release.yaml`

Merging `CHANGELOG/v1.14.0.md` into main triggers the release.
Workflow extracts version from filename via regex. Creates or
checks out release branch. Creates real tag + `test/` tag.
Builds draft GitHub release with all artifacts.

`CHANGELOG/OWNERS` restricts approvals to `cluster-api-release-lead`.
Only the release team can merge the file that triggers release.

**MCN**: Best release trigger pattern found. Turns release into a
standard PR review workflow. Auditable, reviewable, no manual
tagging.

### 12. Fake-Tag Release Smoke Testing

File: `.github/workflows/weekly-test-release.yaml`

Daily cron creates `v9.9.9-fake` tag, runs `make release`. Tests
across main + 2 release branches. Catches release tooling rot
before actual release day.

**MCN**: Cheap insurance. Adopt once release tooling exists.

### 13. Dependabot Auto-Fix (Regenerate Generated Code)

File: `.github/workflows/pr-dependabot.yaml`

On push to `dependabot/**` branches: runs `make generate-modules`
and `make generate`, commits result as `dependabot[bot]`. Solves the
universal problem of stale generated code in Dependabot PRs.

Strategic ignores: controller-runtime, k8s.io/*, cel-go, kind,
kustomize (all need coordinated manual upgrades).

**MCN**: Essential for any Go project with code generation +
Dependabot.

### 14. golangci-lint — 470 Lines, 37 Linters

File: `.golangci.yml`

Notable unique patterns:

- `forbidigo` blocks `ctrl.NewControllerManagedBy`, requires
  wrapper `capicontrollerutil.NewControllerManagedBy` instead
- `ginkgolinter` with `forbid-focus-container: true` (prevents
  accidentally committed `FDescribe`/`FIt`)
- `godox` blocks FIXME keywords (forces cleanup before merge)
- `importas` with `no-unaliased: true` + 35 enforced aliases
- `nolintlint` with `require-specific: true` — no bare `//nolint`

Also runs KAL as custom golangci-lint plugin (same Gateway API
pattern).

**MCN**: The `forbidigo` wrapper enforcement, `forbid-focus-container`,
and `no-unaliased` import enforcement are all worth adopting.

### 15. Import Restrictions on API Packages

File: `api/.import-restrictions`

API packages cannot import `internal` packages or
`controller-runtime`. Enforced by `import-boss` from
k8s.io/code-generator. Prevents coupling APIs to implementation.

**MCN**: Critical if MCN's API types will be imported by
downstream consumers.

### 16. PR Title Emoji Prefixes

File: `.github/workflows/pr-verify.yaml`

PR titles must start with emoji prefixes: warning (breaking),
sparkles (feature), bug (bugfix), book (docs), rocket (release),
seedling (infra/tests). Both emoji and `:emoji_code:` accepted.

**MCN**: Fun alternative to conventional commits. Makes PR lists
scannable at a glance.

### 17. E2E Framework Published as Go Package

CAPI's `test/framework/` is a reusable E2E framework that
downstream providers import. Uses `ClusterProxy` interface
abstraction. YAML-based E2E config with go.mod version resolution
(`{go://sigs.k8s.io/cluster-api@v1.11}`).

**MCN**: If MCN needs a reusable test framework, this is the
reference architecture.

### 18. golangci-lint Version from Workflow YAML

Makefile extracts lint version from the CI workflow file:
`$(shell cat .github/workflows/pr-golangci-lint.yaml | grep ...)`

Ensures local and CI linting use identical versions without
maintaining the version in two places.

**MCN**: Elegant single source of truth for tool versions.

## Crossplane (crossplane/crossplane) — Key Findings

### 19. PR Checklist Enforcement (Deep Dive)

File: `.github/workflows/pr.yml`

`mheap/require-checklist-action@v2`. 7 mandatory checklist items
in PR template. Skips for `crossplane-renovate[bot]`. Triggers on
opened, edited, synchronize.

**MCN**: 5 lines of config, blocks PRs with unchecked items.

### 20. golangci-lint `default: all` with Documented Disables

File: `.golangci.yml` (289 lines)

Starts from `default: all` (every linter enabled) then explicitly
disables ~25 linters, each with a detailed rationale comment.

Notable:

- `depguard` bans testify, ginkgo, gomega in test files (forces
  stdlib testing)
- `interfacebloat` max 5 methods per interface
- `goconst` min 5 occurrences before flagging

**MCN**: The `default: all` approach gets new linters automatically
as golangci-lint adds them. Every disable has documented reasoning.
Consider this vs Submariner's explicit enable list.

### 21. Nix as Sole Build System

File: `flake.nix` (248 lines)

No Makefile. Everything through Nix flakes:

- `nix build` — full release (7 platforms, OCI images, Helm chart)
- `nix flake check` — all CI checks
- `nix develop` — dev shell with all tools

Docker wrapper `nix.sh` for developers without Nix. Cachix binary
cache for pre-built outputs.

**MCN**: Impressive but heavy adoption cost. The hermetic build
guarantees are real but Nix has a steep learning curve. SKIP for
MCN — use Makefile + pinned tool versions instead.

### 22. Separate API Module

`apis/` directory is its own Go module with separate `go.mod`.
Consumers import API types without pulling in the full dependency
tree.

**MCN**: Good practice if MCN's CRD types will be imported by
other projects.

### 23. OSS-Fuzz in Every CI Run

5-minute fuzz runs on every CI run via `google/oss-fuzz`. Catches
parsing bugs continuously.

**MCN**: If MCN handles untrusted network input, register with
OSS-Fuzz for continuous fuzzing.

### 24. Backport — Dual Trigger Pattern

Label-driven auto-backport on merge + manual `/backport` slash
command fallback. `korthout/backport-action` handles merge commits
and multi-commit PRs.

**MCN**: Clean, well-tested pattern. Adopt once release branches
exist.

### 25. No Supply Chain Signing

Notable gap: no cosign, SBOM, or SLSA despite being a CNCF
Graduated project. MCN can do better from day one.

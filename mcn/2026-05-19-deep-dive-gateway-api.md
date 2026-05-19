---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, gateway-api, kal, cel, conformance, crdify]
---

# Deep Dive: Gateway API Repo Automation

File-level deep dive of kubernetes-sigs/gateway-api. 28 files read.
Gateway API is the K8s networking standard MCN will likely implement,
making this the most directly relevant reference.

## Key Findings for MCN

### 1. KAL Integration — The Reference Implementation

Files: `.custom-gcl.yml`, `.golangci-kal.yml`, `.github/workflows/kal.yml`

Three-piece setup:

**`.custom-gcl.yml`** — builds custom golangci-lint binary with KAL
plugin baked in:

```yaml
version: v2.8.0
name: golangci-kube-api-linter
destination: ./bin
plugins:
- module: 'sigs.k8s.io/kube-api-linter'
  version: 'v0.0.0-20260423112246-3fa174937a6b'
```

**Workflow** builds it by running `golangci-lint custom -v` then
invoking the custom binary with `GOOS=js GOARCH=wasm` (skips
compilation of non-API code).

**15 checks enabled**: defaultorrequired, defaults,
duplicatemarkers, jsontags, nodurations, nofloats, nomaps,
nonullable, nophase, noreferences, notimestamp,
optionalorrequired, ssatags, statussubresource, uniquemarkers

**13 checks disabled with documented reasons**:

- `conditions` — would change condition merging, unknown impact
- `integers` — GA APIs already use non-int32/int64
- `maxlength/minlength` — GA fields lack these, would break
- `nobools` — "controversial"
- `optionalfields` — "controversial" (pointers for optional)
- `requiredfields` — omitempty tags
- `statusoptional` — "breaking change for RouteParentStatus"

**MCN**: Copy this exact setup. MCN is starting fresh so we can
enable MORE checks than Gateway API can (they have GA constraints
we don't). Enable `nobools`, `optionalfields`, `requiredfields`,
`integers` from day one — these are only controversial for
existing APIs, not new ones.

### 2. crdify — CRD Breaking Change Detection

File: `hack/verify-crdify.sh`

Uses `sigs.k8s.io/crdify` to detect CRD breaking changes. Compares
each CRD YAML in `config/crd/standard/` against the base ref
(defaults to `main` or Prow's `PULL_BASE_SHA`). Skips new CRD files.
Can be set to warn-only via `CRDIFY_ENFORCE=false`.

**This replaces go-apidiff for CRD projects.** go-apidiff checks Go
API surface; crdify checks the serialized CRD YAML schema. For an
operator project, the CRD schema IS the API contract — not the Go
types.

**MCN**: Adopt crdify instead of (or alongside) go-apidiff. crdify
catches breaking changes at the CRD level where they actually
matter to users. Run it in CI on every PR that touches API types.

### 3. CEL Validation Testing Across K8s Versions

Files: `.github/workflows/crd-validation.yml`, `tests/cel/main_test.go`

Matrix of 5 K8s versions x 2 CRD channels = 10 test combinations:

- K8s: v1.36.0, v1.35.0, v1.34.1, v1.33.0, v1.32.0
- Channels: standard, experimental

Uses controller-runtime envtest with
`DownloadBinaryAssetsVersion: k8sVersion` to get the right API
server binary for each version. Handles cross-version compatibility
(e.g., wording change in k8s v1.32 where "more" became "longer" in
CEL error messages).

Three test packages: `./tests/cel`, `./tests/crd`, `./tests/vap`

**MCN**: Adopt once MCN CRDs have CEL validation rules. Test against
the K8s version range OCP supports (currently ~3 minor versions).

### 4. Conformance Test Framework

Files: `conformance/conformance.go`, `conformance/utils/suite/`

7 profiles: GATEWAY-HTTP, GATEWAY-TLS, GATEWAY-TCP, GATEWAY-UDP,
GATEWAY-GRPC, MESH-HTTP, MESH-GRPC

145 test files in `conformance/tests/`, each paired with YAML
manifests. Can compile as standalone binary for distribution.

Implementations register via flags:

```bash
./conformance-bin --gateway-class=mcn \
  --conformance-profiles=GATEWAY-HTTP,MESH-HTTP \
  --supported-features=... \
  --report-output=report.yaml
```

**MCN**: If MCN implements Gateway API, run the conformance suite
as part of e2e CI. The conformance report can be published as a
release artifact to demonstrate spec compliance.

### 5. Release Artifact Verification

File: `.github/workflows/verify-release-artifacts.yml`

On every GitHub release publish:

1. Downloads `standard-install.yaml` from the release
2. Downloads source tarball for that tag
3. Builds install YAML from source
4. Normalizes copyright years
5. `diff -u` to verify exact match

Fails if release artifact diverges from what source would produce.

**MCN**: Adopt this pattern. Ensures release artifacts are
reproducible from source. Catches stale or tampered release files.

### 6. Custom CRD Generator with Channel Tags

File: `tools/generator/main.go`

Custom CRD generator built on controller-tools. Generates CRDs for
both `standard` and `experimental` channels from same source using
description tags:

- `<gateway:experimental>` — strips fields from standard channel
- `<gateway:validateIPAddress>` — adds oneOf IP validation schema
- `<gateway:{channel}:validation:Enum=...>` — channel-specific enums
- `<gateway:{channel}:validation:XValidation:...>` — channel-specific
  CEL rules

**MCN**: If MCN needs dev-preview vs GA feature gating in CRDs,
this channel-based generation pattern is the reference. Otherwise
standard controller-gen suffices.

### 7. API Documentation Generation

Files: `hack/docsy/generate.sh`, `crd-ref-docs.yaml`

Uses `elastic/crd-ref-docs` with custom markdown templates. Runs
against multiple release branches (release-1.4, release-1.5, main).
Fetches API source via `git archive` per branch. Generates separate
docs for standard and experimental APIs.

**MCN**: Adopt elastic/crd-ref-docs for API reference docs. The
custom template pattern allows MCN-specific formatting.

### 8. Verification Script Pattern

File: `hack/verify-all.sh`

Auto-discovers and runs all `hack/verify-*.sh` scripts, collecting
failures. Individual scripts:

- `verify-codegen.sh` — runs `make generate`, checks git status
- `verify-golint.sh` — Docker-based golangci-lint
- `verify-crdify.sh` — CRD breaking changes

**MCN**: Adopt the `verify-all.sh` auto-discovery pattern. Makes
adding new verification checks trivial — just add a new
`hack/verify-*.sh` script.

### 9. golangci-lint — Notable Choices

File: `.golangci.yml`

28 linters. Notable:

- `gomodguard` blocks `io/ioutil` (deprecated)
- `govet shadow` analysis enabled
- Formatters: `gofumpt` + `goimports` with local prefix
- Separate KAL config (`.golangci-kal.yml`) for API-specific linting

### 10. Notable Absences

- No go-apidiff (uses crdify instead for CRD-level checks)
- No Renovate (uses Dependabot with grouping)
- No cosign/SLSA (images built via Google Cloud Build)
- No harden-runner

## Top MCN Takeaways

1. **crdify > go-apidiff** for CRD-focused projects
2. **KAL with MORE checks enabled** than Gateway API (we start fresh)
3. **CEL testing across K8s versions** via envtest matrix
4. **Release artifact verification** ensures reproducibility
5. **verify-all.sh auto-discovery** for extensible verification
6. **elastic/crd-ref-docs** for API documentation
7. **Conformance test framework** if implementing Gateway API

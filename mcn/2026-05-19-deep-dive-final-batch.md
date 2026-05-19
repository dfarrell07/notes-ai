---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, kong, argo-workflows, olm, vpa, kustomize, buildpacks, network-policy-api]
---

# Deep Dive: Final Batch (#44-50)

Kong, Argo Workflows, OLM completed. VPA, Kustomize, Buildpacks,
Network Policy API results pending.

## Kong Gateway — Key Findings

### 1. ast-grep Custom Lint Rules with Mandatory Tests

Files: `sgconfig.yml`, `.ci/ast-grep/rules/*.yml`

Language-agnostic AST-level linting for Lua code. Each rule must
have a matching test file with valid + invalid cases. CI validates
test coverage per rule. Installed with SHA256 checksum verification.

**MCN**: ast-grep works for any language including Go. Custom AST
rules are more precise than regex-based linting.

### 2. Runtime-Statistics-Driven Test Scheduling

Files: `.github/workflows/build_and_test.yml`,
`Kong/gateway-test-scheduler` action

Dynamically partitions tests across N runners based on historical
runtime data stored via `Kong/gh-storage`. Falls back to static
mode on retry attempts.

**MCN**: Uses actual telemetry to optimize test distribution.
Most projects use static splitting.

### 3. Externalized CI Matrix YAML Files

Files: `.github/matrix-full.yml`, `.github/matrix-commitly.yml`

Full matrix (14 packages, 3 images, 14 releases) vs reduced
per-commit matrix. Selected via `FULL_RELEASE` flag. Makes matrix
changes reviewable independently.

**MCN**: Separates platform support from workflow logic.

### 4. Binary Manifest Verification

File: `scripts/explain_manifest/main.py`

Python tool extracts and verifies ELF binary properties, file
lists, shared library deps from both .deb/.rpm packages and Docker
images against expected baselines per platform.

## Argo Workflows — Key Findings

### 5. Feature Description Enforcement on feat PRs

File: `.github/workflows/pr-feature.yaml`

PRs with `feat(...)` titles must include a new
`.features/pending/*.md` file with structured template:
Description, Authors, Component, Issues. `make features-validate`
validates content. Released features move to `released/`.

**MCN**: Enforcing user-facing feature documentation at PR time.
Enables automated release note generation from structured data.

### 6. devenv.nix for Development with process-compose

File: `devenv.nix`

Nix-based development environment. Defines full stack (controller,
argo-server, ui) with dependencies. Uses `process-compose` for
orchestration.

### 7. /retest with Failed-Jobs-Only Re-Run

File: `.github/workflows/retest.yaml`

`/retest` comment finds latest CI run for PR's HEAD SHA, re-runs
only failed jobs. More targeted than full workflow re-runs.

### 8. Centralized K8s Version Bounds

File: `hack/k8s-versions.sh`

`K8S_VERSIONS[min]` and `K8S_VERSIONS[max]` as single source of
truth for CI, devcontainer, and docs.

## OLM — Key Findings

### 9. Two-Tier Tool Management: bingo + tools.go

Files: `.bingo/Variables.mk`, `tools.go`

- **bingo**: Tools orthogonal to project (lint, helm, kind) — each
  gets own `.mod` file, compiled to versioned binary
- **tools.go**: Tools that must track k8s library versions (code
  generators, mocks)

Explicit rationale documented in Makefile comments.

**MCN**: Most well-reasoned tool management found. Clear
separation with documented decision criteria.

### 10. Go AST-Based Ginkgo Test Splitter

File: `test/e2e/split/main.go`

Custom Go program parses Go AST for `ginkgo.Describe` calls with
`Label()` arguments. Extracts labels, deterministically splits
into N chunks. Generates `--label-filter` expressions for parallel
Ginkgo execution.

**MCN**: More precise than package-based splitting for Ginkgo
test suites.

### 11. Multi-Cluster E2E per Runner

File: `.github/workflows/e2e-tests.yml`

Creates multiple Kind clusters per matrix job (E2E_NODES=2), each
with separate kubeconfig. Ginkgo `-nodes` flag distributes across
clusters.

**MCN**: Solves the problem where tests mutate cluster state in
incompatible ways.

### 12. Separated [FLAKE] Tests

OLM explicitly labels flaky tests with `[FLAKE]`. Main E2E skips
them. Separate `e2e-flakes` job runs only flakes (single node for
stability). Keeps flakes from blocking merges while tracking them.

**MCN**: Clean pattern for managing known flaky tests without
hiding them.

### 13. go-verdiff — Go Version Drift Detection

File: `.github/workflows/go-verdiff.yaml`

Detects any Go version changes in `.mod` files compared to base
branch. Fails if root `go.mod` version increases. Label
`override-go-verdiff` can override.

**MCN**: Prevents accidental Go version bumps.

### 14. AGENTS.md — 350 Lines of AI Context

File: `AGENTS.md`

Architecture, directory structure, CRDs, state machines,
anti-patterns, navigation tips. Most comprehensive AI agent
context file found across all 50 projects.

## VPA (Vertical Pod Autoscaler) — Key Findings

### 15. Dead Code Elimination Verification

File: `hack/verify-deadcode-elimination.sh`

Runs `whydeadcode` against all component binaries. Compiles with
`-ldflags=-dumpdep`, pipes dependency graph into whydeadcode.
Fails CI if dead code paths detected. First project with
automated dead code detection as CI gate.

**MCN**: Apply to MCN operator, gateway, agent binaries.

### 16. VPA Benchmark Suite with KWOK

File: `test/benchmark/main.go`

Uses KWOK to simulate large clusters. Profiles from "small"
(25 VPAs, 50 pods) to "xxlarge" (1000 VPAs, 2000 pods).
Port-forwards to scrape Prometheus metrics during benchmarks.

## Kustomize — Key Findings

### 17. new-from-rev for Progressive Linting

File: `.golangci.yml`

`issues.new-from-rev: c94b5d8f2` — enforces strict linters only
on new code. Combined with `enable-all: true`. Existing code
remains unlinted. Practical for adopting aggressive linting in
existing codebases.

**MCN**: MCN starts fresh so can use `enable-all` without
new-from-rev. But useful pattern for retrofitting linting.

### 18. Module Count Sanity Check

File: `for-each-module.sh`

48+ Go modules in `go.work`. Script has expected count check —
fails if modules silently added or removed.

**MCN**: Simple guard for multi-module repos.

### 19. go-apidiff with Auto-Issue on Failure

File: `.github/workflows/apidiff.yml`

Runs `go-apidiff` on PRs. On push-to-master failure, auto-creates
GitHub issue via `nashmaniac/create-issue-action`.

**MCN**: go-apidiff + auto-issue creation on main branch failures.

## Buildpacks (pack) — Key Findings

### 20. Cross-Version Compatibility Matrix

File: `.github/workflows/compatibility.yml`

Tests all valid combos of `{current, previous}` for CLI, builder,
and lifecycle. Explicit exclude rules for impossible combinations.

**MCN**: Test current-broker-with-previous-gateway and vice versa.

### 21. Benchmark Tracking with 200% Regression Alert

File: `.github/workflows/benchmark.yml`

`benchmark-action/github-action-benchmark` stores results on
gh-pages. Auto-alerts with commit comment + failure if perf
regresses > 200%. Tags specific maintainer teams.

### 22. 5-Channel Release Distribution

Files: `delivery-homebrew.yml`, `delivery-chocolatey.yml`,
`delivery-ubuntu.yml`, `delivery-archlinux.yml`,
`delivery-release-dispatch.yml`

Triggers on `release:released`. Pushes to Homebrew, Chocolatey,
Ubuntu PPA, AUR, Docker Hub. Notifies 4 downstream repos via
`repository-dispatch`.

## Network Policy API — Key Findings

### 23. Conformance Report Framework

Files: `conformance/conformance_profile_test.go`,
`conformance/reports/v0.1.2/ovn-kubernetes.yaml`

Formal conformance framework with typed `ConformanceReport` YAML:
implementation metadata, profile results (standard/experimental),
statistics. Implementation reports stored in-repo.

**MCN**: Most directly relevant for MCN's networking conformance.
Generate structured YAML reports from connectivity tests.

### 24. CRD Validation as Portable Test Binary

File: `pkg/crdtest/crd_test.go`

`//go:embed testdata/*/*` embeds valid/invalid fixtures.
`go test -c` produces standalone portable CRD validation binary.

**MCN**: If MCN distributes CRDs, provide a portable validation
binary.

### 25. Network Policy Enhancement Proposals (NPEPs)

Directory: `npeps/`

In-repo enhancement proposal process with numbered proposals.
Stored as markdown, copied to docs site.

**MCN**: Similar to Longhorn LEPs. Lightweight KEP-like process.

## All 50 Deep Dives Complete

Projects surveyed:

1-10: Cilium, Gateway API, Istio, cert-manager, etcd, Argo CD,
Kyverno, Flux, Cluster API, Crossplane

11-20: Prometheus Operator, MetalLB, Contour, Dapr, KEDA,
Knative, Linkerd, Velero, External Secrets, Strimzi

21-30: Envoy Gateway, Falco, Rook, OPA, Tekton, CoreDNS, Thanos,
Loki, Jaeger, Harbor

31-40: KubeVirt, Volcano, Chaos Mesh, Flagger, Karmada, Longhorn,
Keptn, KubeEdge, Emissary-Ingress, Telepresence

41-50: Traefik, Ingress-NGINX, Helm, Kong, Argo Workflows, OLM,
VPA, Kustomize, Buildpacks, Network Policy API

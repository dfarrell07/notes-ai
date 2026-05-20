---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [proposal, tooling, ci, linting, testing, security, release]
---

# MCN Tooling Proposal — Consolidated from 50 Project Deep Dives

Definitive tooling proposal for the MCN operator repo. Every tool
evaluated against 50 K8s/CNCF project deep dives plus the
Submariner automation review. Sorted by adoption likelihood within
each category.

## 1. Go Source Linting

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| golangci-lint v2 | 60+ linters, v2 config format | Yes | 1 |
| kube-api-linter (KAL) | CRD API convention enforcement | Yes | 1 |
| goheader | Apache-2.0 license header enforcement | Yes | 1 |
| importas | Enforce K8s import aliases (corev1 etc.) | Yes | 1 |
| modernize | Suggest modern Go idioms | Yes | 1 |
| funcorder | Constructor/method ordering | Yes | 1 |
| recvcheck | Consistent pointer/value receivers | Yes | 1 |
| iface | Detect incorrect interface use | Yes | 1 |
| depguard/gomodguard | Ban deprecated packages | Yes | 1 |
| sloglint | Structured logging conventions | Yes | 2 |
| promlinter | Prometheus metrics naming | Yes | 2 |
| exhaustive | Exhaustive switch coverage | Yes | 2 |
| bidichk | Unicode bidi control chars (supply chain) | Yes | 2 |
| faillint | Import policy enforcement (stronger) | Consider | 2 |
| forbidigo | Ban specific functions/calls | Yes | 2 |

**Config approach**: Start from Submariner's 248-line config.
Change goheader to "MCN project". Remove Submariner-specific
exclusions. Add the new linters above. Use `default: none` with
explicit enable list (not `default: all`). Separate `formatters`
section with `gci`, `gofmt`, `gofumpt`, `goimports`.

**KAL config**: Build custom golangci-lint binary via
`.custom-gcl.yml`. Enable ALL 28 checks (MCN starts fresh — no
GA API constraints like Gateway API). Scope to `api/` directory.

## 2. Non-Go Linting

| Tool | Content Type | Adopt? | Phase |
| --- | --- | --- | --- |
| markdownlint-cli2 | Markdown | Yes | 1 |
| yamllint | YAML | Yes | 1 |
| shellcheck | Shell scripts | Yes | 1 |
| hadolint | Dockerfiles | Yes | 1 |
| actionlint | GitHub Actions workflows | Yes | 1 |
| zizmor | GitHub Actions security | Yes | 1 |
| gitlint | Commit messages | Yes | 1 |
| kubeconform | K8s manifest validation | Yes | 1 |
| kube-linter | K8s manifest security (40+ checks) | Yes | 1 |
| lychee | Markdown link checking | Yes | 1 |
| shfmt | Shell script formatting | Yes | 2 |
| checkmake | Makefile linting | Consider | 3 |
| IBM/tekton-lint | Tekton pipeline YAML | Consider | 4 |

## 3. Security Scanning

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| govulncheck | Go vulnerability scanning (SARIF) | Yes | 1 |
| CodeQL | SAST via GitHub Code Scanning | Yes | 1 |
| OSSF Scorecard | Supply chain security assessment | Yes | 1 |
| zizmor | GHA workflow security | Yes | 1 |
| SHA-pinned actions check | Verify all GHA refs use SHA pins | Yes | 1 |
| dependency-review-action | Block PRs with known-vulnerable deps | Yes | 1 |
| Anchore/grype | Container vulnerability scanning | Yes | 2 |
| TruffleHog | Verified secrets scanning in PR diffs | Yes | 2 |
| harden-runner | Network egress control for GHA | Yes | 2 |
| GODEBUG security flags | Prevent tar/zip path traversal | Yes | 1 |
| Cosign keyless signing | Image + Helm chart signing | Yes | 3 |
| Syft | SBOM generation (SPDX) | Yes | 3 |
| SLSA provenance | Build provenance attestations | Yes | 3 |
| OSV-Scanner | Dual-mode vuln scanning (PR + weekly) | Consider | 2 |
| Trivy multi-branch | Scan all supported release branches | Yes | 3 |

## 4. Testing

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| Ginkgo/Gomega | BDD test framework + matchers | Yes | 1 |
| envtest | Local API server for controller tests | Yes | 1 |
| KIND clusters | E2E test infrastructure | Yes | 2 |
| Codecov | Coverage reporting with PR comments | Yes | 2 |
| Go native fuzzing | Webhook validation fuzz tests | Yes | 2 |
| go-ordered-test | Test isolation detector (weekly) | Yes | 2 |
| go-stress-test | Flaky test detector (nightly) | Yes | 2 |
| overcover | Coverage ratchet (only goes up) | Yes | 2 |
| go-test-split-action | Integration test parallelization | Consider | 3 |
| gotesplit | E2E parallelization via artifacts | Consider | 3 |
| CIFuzz/OSS-Fuzz | Continuous fuzzing (10-20 min/PR) | Consider | 3 |
| upgrade E2E | N-1 to N version upgrade testing | Yes | 3 |
| system validation | Deployment correctness script | Yes | 3 |
| version skew testing | N-1 compat between components | Consider | 3 |
| binary size tests | Prevent binary bloat (min/max MB) | Consider | 3 |
| KWOK + k6 | Controller performance testing | Consider | 3 |
| CANNIER | ML-informed flake detection | Consider | 3 |
| `-shuffle=on` test flag | Catch ordering-dependent tests | Yes | 1 |
| `go mod tidy -diff` | Clean go.mod drift check (Go 1.26+) | Yes | 1 |

## 5. CRD and API Validation

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| controller-gen | CRD + DeepCopy generation | Yes | 1 |
| CI codegen diff check | Verify generated code is committed | Yes | 1 |
| crdify | CRD breaking change detection | Yes | 2 |
| go-apidiff | Go API backward compatibility | Yes | 2 |
| CEL validation matrix | Test CEL across K8s versions | Yes | 2 |
| elastic/crd-ref-docs | Auto-generated API reference docs | Yes | 3 |
| CRD JSON Schema release | IDE validation of custom resources | Consider | 3 |
| operator-sdk bundle validate | OLM bundle validation | Yes | 4 |
| CRD drift verification | Diff CRDs across Helm/kustomize/raw | Yes | 3 |

## 6. Container Images

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| ko | Go image builds without Dockerfile | Yes (upstream) | 1 |
| Multi-stage Dockerfile | UBI9/distroless production images | Yes (downstream) | 2 |
| Cosign keyless signing | Image signing via OIDC | Yes | 3 |
| Syft SBOM | Generate + attach SBOMs to images | Yes | 3 |
| SLSA provenance | Per-image build attestations | Yes | 3 |
| Trivy multi-branch scan | Scan all release branches weekly | Yes | 3 |
| tonistiigi/xx | Cross-compilation (faster than QEMU) | Consider | 3 |

## 7. Release Automation

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| Dependabot | GHA monthly + Go modules weekly | Yes | 1 |
| Per-PR changelog files | Contour pattern, CI-enforced | Yes | 2 |
| release-please | Automated versioning from commits | Yes | 3 |
| GoReleaser | Cross-platform binary builds | Consider | 3 |
| GoReleaser dry-run on config change | Catch release config bugs | Yes | 3 |
| Backport action | Auto cherry-pick on label | Yes | 3 |
| Release artifact verification | Diff release vs source | Yes | 3 |
| Fake release smoke test | Daily `v9.9.9-fake` build | Yes | 3 |
| Renovate | Advanced dep management (if needed) | Consider | 3 |
| Dependabot auto-fix | Regenerate code on dep updates | Yes | 2 |

## 8. CI Workflow Patterns

| Tool/Pattern | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| dorny/paths-filter | Skip jobs based on changed files | Yes | 1 |
| Stale issue/PR cleanup | Close inactive issues/PRs | Yes | 1 |
| Branch enforcement | PRs must target main/release-* | Yes | 1 |
| PR checklist enforcement | Block merge on unchecked items | Yes | 1 |
| Draft-aware test matrix | Minimal CI for draft PRs | Yes | 2 |
| Workflow failure issue tracker | Auto-create issues on CI failure | Yes | 2 |
| Workflow telemetry | CI timing metrics per step | Yes | 2 |
| Trust-based CI parallelism | Faster CI for trusted contributors | Consider | 3 |
| Composite result pattern | Aggregate matrix into single check | Yes | 2 |
| Write-on-merge/read-on-PR cache | Warm caches for all PRs | Yes | 2 |
| Comment-triggered E2E | `/run-e2e` with team membership gate | Consider | 3 |

## 9. Developer Experience

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| CLAUDE.md | Claude Code project conventions | Yes | 1 |
| AGENTS.md | AI agent instructions (tool-agnostic) | Yes | 1 |
| Makefile | Standalone build system | Yes | 1 |
| `.github/env` | Centralized tool version pinning | Yes | 1 |
| Lefthook | Pre-commit hooks (Go native, parallel) | Yes | 2 |
| devcontainer.json | Consistent dev environment | Consider | 2 |
| SECURITY-INSIGHTS.yml | OpenSSF security metadata | Yes | 2 |
| Inclusive language linting | woke on changed files | Consider | 2 |
| echo.% Makefile introspection | Debug Make variables | Yes | 1 |

## 10. AI-Powered PR Review (GitHub Actions)

| Workflow | What It Reviews | Adopt? | Phase |
| --- | --- | --- | --- |
| ai-security-review | RBAC, privilege escalation, secrets | Yes | 1 |
| ai-rbac-review | Detailed RBAC change analysis | Yes | 1 |
| ai-release-notes | Suggest release notes for PRs | Yes | 1 |

All use `anthropics/claude-code-action@v1` with
`ANTHROPIC_API_KEY` secret. Non-blocking (informational PR
comments only). `use_sticky_comment: true` to avoid spam.
Read-only tools only.

## Key Architectural Decisions

1. **ko for upstream, Konflux Dockerfiles for downstream** — ko
   eliminates upstream Dockerfiles entirely. Downstream requires
   UBI9 for Red Hat builds.

2. **Conventional Commits + release-please** — automates version
   bumps and changelogs from commit messages.

3. **Per-PR changelog files (Contour pattern)** — CI-enforced,
   category-labeled, assembled at release time. Best pattern found
   across 50 projects.

4. **KAL with all 28 checks enabled** — MCN starts fresh, no GA
   API constraints. Enable `nobools`, `optionalfields`,
   `requiredfields`, `integers` from day one.

5. **crdify over go-apidiff for CRD changes** — check serialized
   CRD YAML schema, not just Go API surface.

6. **GHA + Prow split** — GHA for linting/unit/images/AI reviews.
   Prow for real cloud E2E, upgrades, scale testing.

7. **golangci-lint v2 `default: none` with explicit enables** —
   deterministic, no surprise additions from lint upgrades.

8. **`.github/env` for centralized version pinning** — single file
   controls Go, golangci-lint versions across all CI jobs.

9. **OSSF Scorecard from day one** — single workflow, instant
   security visibility and public badge.

10. **Three AI review workflows** — security, RBAC, release notes.
    Non-blocking, confidence-scored, read-only tools.

## Adoption Phases

### Phase 1 — Day One (Before Any Code)

22+ CI linting jobs, pre-commit hooks, AI reviews, governance
workflows. Everything in this phase can be set up with zero Go
code in the repo.

### Phase 2 — First Code Landing

Controller tests (envtest), coverage tracking, code generation CI
checks, fuzzing, dependency management automation, Lefthook.

### Phase 3 — Approaching First Release

Full E2E matrix, upgrade testing, release automation, supply chain
security (Cosign, SLSA, SBOMs), multi-arch builds, backport
automation.

### Phase 4 — Downstream / Konflux Onboarding

Konflux Dockerfiles, Tekton pipelines, Enterprise Contract, RPM
lockfiles, OLM bundles, FBC catalogs, stage/prod release workflow.

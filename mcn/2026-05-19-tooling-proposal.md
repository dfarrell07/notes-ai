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

## Go Linting Tools — Detailed Pro/Con Analysis

Research from web searches, GitHub issues, and adoption data across
50 K8s/CNCF projects.

### golangci-lint v2

**Current version**: v2.12.2 (May 2026). The dominant Go linting
aggregator — no real alternative exists.

**Pro**: v2 has cleaner config (separate formatters section,
`linters.default`, human-readable exclusion presets). `golangci-lint
migrate` converts v1 configs. Active release cadence (12 minor
releases in 14 months). Major projects migrated: Coder, Antrea,
GitLab, Kubebuilder.

**Con**: v1 configs completely incompatible — hard break, not
deprecation. Migration tool has bugs (forbidigo produces broken
YAML, comments lost). IDE breakage reported for nvim-lint and VS
Code Go extension. Default behavior changes silently (exclude-
generated changed from lax to strict). Breaks on some new Go
releases.

**MCN verdict**: Must-have. New project starts with v2 directly —
no migration pain. v1 will not receive new features.

### kube-api-linter (KAL)

**Current version**: Pre-release, no tagged versions (pseudo-
versions like `v0.0.0-20260114...`). kubernetes-sigs project.

**Pro**: Automates mechanical elements of K8s API review. 19+
rules enforcing API conventions (nobools, nophase, integers,
jsontags, optionalorrequired, defaults, statussubresource, nomaps,
requiredfields, commentstart). Adopted by openshift/api,
openshift/hypershift, cluster-api, cluster-api-provider-aws. Being
adopted by upstream kubernetes/kubernetes (PR #131561). Autofix for
some rules. golangci-lint v2 plugin integration.

**Con**: No stable release — still v0.0.0 pseudo-versions. Setup
requires building custom golangci-lint binary via `.custom-gcl.yml`.
False positive management is manual. Not all rules apply to CRDs
(e.g., `nonpointerstructs` says "NOT intended for CRD types").
Only checks Go struct definitions — not controller logic.

**MCN verdict**: Adopt with care. MCN defines CRDs so KAL catches
API convention violations before API review. Enable rules
selectively: jsontags, optionalorrequired, requiredfields, defaults,
statussubresource, nobools first. Skip nonpointerstructs for CRDs.
Accept pre-release risk — backed by kubernetes-sigs.

### goheader

**Current version**: Bundled with golangci-lint (upstream:
denis-tingaikin/go-header).

**Pro**: Native golangci-lint integration — no extra tools. Flexible
templates with const/regexp values. Built-in year variables (YEAR,
YEAR-RANGE, MOD_YEAR). Template file support keeps config clean.

**Con**: Check-only, no auto-fix — need separate tool (addlicense
or SkyWalking Eyes) to actually fix violations. Go-only — doesn't
check shell, Dockerfile, YAML headers. Year handling pitfalls
(`{{ YEAR }}` breaks on new year unless using YEAR-RANGE). Known
bugs: sometimes reports nothing even when headers are wrong
(issue #1446). Build directive conflicts with gofmt.

**MCN verdict**: Use for Go files in golangci-lint pipeline. Pair
with addlicense or SkyWalking Eyes for fixing and multi-language
coverage. Use YEAR-RANGE regex to avoid new-year breakage.

### importas

**Current version**: Bundled with golangci-lint since v1.38.0.
Upstream: julz/importas (last tagged 2022, golangci-lint vendors
newer commit).

**Pro**: Solves real K8s problem — dozens of packages with
colliding final path segments (multiple `v1`, `v1alpha1`). Regex
pattern support (one rule covers all k8s.io/api packages). Autofix
support. `no-unaliased: true` prevents alias drift.

**Con**: Non-deterministic behavior with multiple regex rules
(issue #5218, fixed in recent golangci-lint). Upstream is stale
(last tag 2022). `no-unaliased: true` can be overly broad for
legacy codebases. Viper config limitation prevents natural map
format.

**MCN verdict**: Adopt from day one. New project has no legacy code
so `no-unaliased: true` is safe. Configure regex rules for
standard K8s package patterns. Use golangci-lint v2.12+ to avoid
the non-determinism bug.

### modernize

**Current version**: Part of golang.org/x/tools (v0.40.0 in
golangci-lint v2.12.x). Official Go team tool.

**Pro**: Official Go team backing. Catches concrete outdated
patterns: strings.CutPrefix, slices.Contains, slices.Sort,
maps.Clone, min/max builtins, range-over-int, `any` instead of
`interface{}`, testingcontext, waitgroup. Fixes are designed to be
behavior-preserving. Used by CoreDNS. Active development.

**Con**: Known edge cases with nilness (slices.Clone returns nil
for empty slice, append returns non-nil empty — issue #73557).
Fixes can cause unused imports requiring manual cleanup. Known
panic in stringscut (issue #77451). Requires Go >= 1.21 for most
suggestions.

**MCN verdict**: Enable. New project targets Go 1.26+ so all
suggestions apply. Run with `--fix` during development but review
diffs. The nilness edge cases are documented and the problematic
analyzers have been disabled from automatic application.

### funcorder

**Current version**: v0.6.0 (manuelarte/funcorder). Added to
golangci-lint via PR #5630. New linter (2025).

**Pro**: Configurable: constructor, struct-method, alphabetical,
function checks can be enabled independently. Prevents "where does
this method live?" problem in large controller files. Pure style
enforcer — no false positives in the technical sense.

**Con**: Inherently opinionated — no correctness argument for
function ordering. Teams preferring logical grouping (create/update/
delete together) will fight it. The `alphabetical` option is
controversial and breaks logical grouping. Very new, low adoption
— no major K8s/CNCF projects enable it yet.

**MCN verdict**: Enable defaults only (constructor + struct-method).
Do NOT enable alphabetical — creates friction without benefit.
Skip `function` unless package-level ordering causes confusion.

### recvcheck

**Current version**: Integrated into golangci-lint late 2024.
Maintained at raeperd/recvcheck.

**Pro**: Catches real bugs — mixing pointer and value receivers on
a type with mutex is a data race. Smart built-in exclusions for
marshal methods (MarshalJSON, MarshalText, etc.). Directly
implements a rule from Go's CodeReviewComments. Configurable
exclusions via `struct_name.method_name` patterns.

**Con**: False positives when implementing interfaces requiring
specific receiver types. Alternative linter smrcptr exists (slight
ecosystem fragmentation). Some developers intentionally mix
receivers for `String()` on fmt.Stringer (handled by exclusions).

**MCN verdict**: Enable. One of the highest-value linters. K8s
operators have many structs with methods — reconcilers, webhooks,
controllers. Mixed receivers are a real bug source. Add `*.String`
to exclusions if using fmt.Stringer with value receivers.

### iface

**Current version**: v1.4.2 in golangci-lint. Maintained at
uudashr/iface. 4 analyzers: identical, unused, opaque, unexported.

**Pro**: `identical` analyzer (default) catches duplicate interface
definitions within a package — low noise. `opaque` catches
premature abstraction. Supports `//iface:ignore` directives.

**Con**: `unused` analyzer has known false positives — only checks
within defining package, misses cross-package consumption (standard
Go pattern). `opaque` false-positives on intentional interface
returns for testability (common in K8s operators returning
`client.Client`). During golangci-lint integration, recommendation
was to disable `unused` and `opaque` by default due to noise.

**MCN verdict**: Enable with `identical` only (the default).
Optionally add `unexported`. Do NOT enable `unused` or `opaque` —
K8s operators use cross-package interfaces and interface returns
for testability idiomatically, causing high false positive rates.

### depguard (vs gomodguard)

**Current version**: depguard v2 stable in golangci-lint.
gomodguard has new `gomodguard_v2` as of May 2026.

**Difference**: depguard operates per-file per-import (can block
`log` in production but allow in tests). gomodguard operates per-
module (go.mod level, version constraints). They work at different
levels and can run simultaneously.

**Pro (depguard)**: Fine-grained per-file rules with deny messages.
Three matching modes (original, strict, lax). Used by Cilium with
both depguard AND gomodguard.

**Con (depguard)**: v1-to-v2 config migration was painful (silent
breakage in golangci-lint v1.53). Default behavior blocks all non-
stdlib imports if enabled without config. Slower than gomodguard
(reads every file).

**MCN verdict**: Use depguard with `lax` mode. v1/v2 migration
irrelevant for new project. Typical deny rules: `io/ioutil`,
`math/rand$` -> `math/rand/v2`, `log$` -> `log/slog`. Add
gomodguard only if you need module-level version pinning.

### sloglint

**Current version**: v0.11.1 (July 2025). Supports autofix.
Maintained at go-simpler/sloglint.

**Pro**: Catches real runtime errors statically — slog silently
produces `BADKEY` entries on mismatched key-value pairs. Autofix
support. Highly configurable. Cilium adopted it (February 2025)
with thorough configuration.

**Con**: Only useful if you actually use `log/slog`. K8s operator
ecosystem norm is logr + zap (Operator SDK default). Enabling
implies commitment to slog as your logging API. Some rules are
very opinionated (e.g., `args-on-sep-lines`).

**Nuance**: logr and slog are bridgeable since logr v1.3+ via
`logr.FromSlogHandler()`. You can use slog as the API while
controller-runtime uses logr internally.

**MCN verdict**: Depends on logging strategy. If using Operator
SDK default (logr + zap): skip. If choosing slog as application
API (bridged to logr): enable with `no-mixed-args` and
`no-raw-keys` at minimum.

### promlinter

**Current version**: v0.3.0 (86 stars). Uses upstream
prometheus/client_golang promlint library.

**Pro**: Catches metrics naming violations that Prometheus
community conventions require. Getting naming right from day one
avoids painful breaking-change renames later. Selective disable
per check. Silent skip for unparseable metrics (no noise from
dynamic construction).

**Con**: Cannot handle metrics names built from variables/constants
(extremely common in K8s operators where namespaces are constants).
These metrics are simply invisible to the linter. Small project
(86 stars), single maintainer.

**MCN verdict**: Enable. Low noise (skips what it can't parse),
catches real naming violations. The cost of wrong metrics naming
in a shipped operator is high.

### exhaustive

**Current version**: 339 stars, 45 releases, actively maintained.

**Pro**: Catches real bugs — adding new enum value silently falls
through to default. Pulumi found it prevented multiple runtime
panics. Rich configuration: `ignore-enum-members`,
`explicit-exhaustive-switch`, `check-generated`. Also checks map
literal exhaustiveness.

**Con**: Known memory issue when running through golangci-lint
(issue #5065). Proto-generated enums are very noisy without
careful config (`check-generated: false`, `ignore-enum-members:
".*_UNSPECIFIED"`). Without `default-signifies-exhaustive: true`,
extremely noisy.

**MCN verdict**: Enable with `default-signifies-exhaustive: true`.
K8s operators define state machine enums (phase, condition types)
— missing a value in a switch is a real bug. Set
`check-generated: false`.

### bidichk

**Current version**: v0.3.3 (March 2025, 42 stars, 0 open issues).

**Pro**: Zero false positive rate — legitimate Go code never
contains BiDi override characters. Negligible CI time (simple
byte scan). Defense against Trojan Source attack (CVE-2021-42574).
Go's compiler doesn't warn about BiDi characters (unlike Rust,
GCC).

**Con**: Attack vector has debatable real-world severity (Rapid7
argues CVSS 9.8 is inflated, calculates 5.6). No known real-world
supply chain attacks via this technique. Very small project (42
stars).

**MCN verdict**: Enable. Zero cost, zero noise. For an open-source
K8s operator consumed by others, this is basic supply chain
hygiene.

### faillint

**Current version**: v1.15.0 (March 2025, 244 stars). Created by
Fatih Arslan. NOT integrated into golangci-lint — standalone only.

**Pro**: Declaration-level granularity (ban specific functions
within a package, e.g., `fmt.{Errorf}`). Suggestion system. Used
by Thanos and Grafana Loki.

**Con**: Not in golangci-lint — requires separate CI step,
separate config, separate nolint directives. depguard + forbidigo
together cover most functionality within golangci-lint.

**MCN verdict**: Skip. Use depguard (package-level) + forbidigo
(function-level) instead — both integrated in golangci-lint.

### forbidigo

**Current version**: v1.6.0, updated to v2.3.1 in golangci-lint.
Actively maintained.

**Pro**: Most powerful function-level enforcement. Regex patterns
match method calls, struct field access, and more. Type-aware mode
(`analyze_types`) handles import aliases. Proven in production at
Cilium (banning ~40 netlink functions) and KubeVirt (banning
Ginkgo focused tests).

**Con**: `analyze_types` has performance impact. Without it,
aliased imports bypass patterns. Regex patterns can be tricky to
get right.

**MCN verdict**: Enable. Start with defaults (banning `fmt.Print`
family) and add operator-specific rules. Useful early bans:
`^fmt\.Print(|f|ln)$` (use structured logging),
`^http\.DefaultClient$` (enforce timeouts). Pair with depguard
for package-level governance.

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

| Tool | What It Does | Adopt? | Phase | Audit Notes |
| --- | --- | --- | --- | --- |
| golangci-lint v2 | 60+ linters, v2 config | Yes | 1 | GPL-3.0 (fine for tooling). Bus factor: 1 |
| kube-api-linter (KAL) | CRD API conventions | Yes | 1 | Apache-2.0. Pre-release (no tags). SIG API Mach |
| goheader | License header enforcement | Yes | 1 | GPL-3.0. Pair with addlicense for fixing |
| importas | K8s import alias enforcement | Yes | 1 | Apache-2.0. Non-determinism fixed in v0.2.0 |
| modernize | Modern Go idioms | Yes | 1 | BSD-3. Official Go team. Known edge cases |
| funcorder | Constructor/method ordering | Yes | 1 | Apache-2.0. New, low adoption. Defaults only |
| recvcheck | Receiver consistency | Yes | 1 | MIT. High value, configure K8s exclusions |
| iface | Interface pollution detection | Yes | 1 | Apache-2.0. `identical` only for operators |
| depguard | Ban deprecated packages | Yes | 1 | GPL-3.0. Stable v2. Use `lax` mode |
| forbidigo | Ban specific functions | Yes | 2 | Apache-2.0. Active. Function-level banning |
| promlinter | Prometheus metrics naming | Yes | 2 | Apache-2.0. Stale (1yr). Low noise |
| exhaustive | Exhaustive switch coverage | Careful | 2 | BSD-2. 18mo release gap. Memory issues |
| sloglint | Structured logging (slog) | If slog | 2 | MPL-2.0. Skip if using logr+zap |
| bidichk | Unicode bidi chars | Drop | - | MIT. Redundant with gosec G116 |
| faillint | Import policy enforcement | Skip | - | BSD-3. depguard+forbidigo covers it |

**Config approach**: Start from Submariner's 248-line config.
Change goheader to "MCN project". Remove Submariner-specific
exclusions. Add the new linters above. Use `default: none` with
explicit enable list (not `default: all`). Separate `formatters`
section with `gci`, `gofmt`, `gofumpt`, `goimports`.

**KAL config**: Build custom golangci-lint binary via
`.custom-gcl.yml`. Enable selectively (jsontags, optionalorrequired,
requiredfields, defaults, statussubresource, nobools first). Scope
to `api/` directory. Accept pre-release risk — SIG API Machinery
backed.

**Changes from initial proposal (post-audit)**:

- **bidichk**: Dropped — redundant with gosec G116 (already enabled
  via golangci-lint's gosec integration)
- **faillint**: Confirmed skip — depguard + forbidigo fully covers
  its features with golangci-lint integration
- **exhaustive**: Downgraded from "Yes" to "Careful" — 18-month
  release gap, memory issues, proto noise. Use
  `explicit-exhaustive-switch` mode
- **KAL**: Adjusted from "enable ALL 28 checks" to "enable
  selectively" based on pre-release stability concerns
- **sloglint**: Clarified as conditional on logging strategy choice

## 2. Non-Go Linting

| Tool | Content Type | Adopt? | Phase | Audit Notes |
| --- | --- | --- | --- | --- |
| markdownlint-cli2 | Markdown | Yes | 1 | MIT. 629 stars. David Anson (Microsoft). Node.js |
| yamllint | YAML | Yes | 1 | GPL-3.0. 3.3K stars. Python. Standard |
| shellcheck | Shell scripts | Yes | 1 | GPL-3.0. 37K+ stars. Undisputed. Haskell binary |
| hadolint | Dockerfiles | Yes | 1 | GPL-3.0. 12K stars. Integrates ShellCheck |
| actionlint | GHA correctness | Yes | 1 | MIT. 3.5K stars. Go binary. Pair with zizmor |
| zizmor | GHA security | Yes | 1 | MIT. 5K stars. Rust. 24 security rules |
| conform (or PR title check) | Commit/PR linting | Yes | 1 | MPL-2.0. 520 stars. Go. K8s-native. See below |
| kubeconform | K8s manifest schemas | Yes | 1 | Apache-2.0. 3K stars. Replaces kubeval |
| kube-linter | K8s manifest security | Yes | 1 | Apache-2.0. 3.4K stars. Red Hat/StackRox |
| lychee | Link checking | Yes | 1 | Apache-2.0. 3.6K stars. Rust. Replaces md-link-check |
| shfmt | Shell formatting | Yes | 2 | BSD-3. 8.8K stars. mvdan (Go contributor) |
| checkmake | Makefile linting | Consider | 3 | MIT. 1.2K stars. Small but active |
| IBM/tekton-lint | Tekton YAML | Skip | - | Apache-2.0. 31 stars. Barely maintained. Use kubeconform |

**Changes from initial proposal (post-audit)**:

- **gitlint**: Replaced with **siderolabs/conform** (Go, 520
  stars, MPL-2.0, K8s-native). gitlint is unmaintained (3yr gap,
  seeking co-maintainers). conform provides Conventional Commits +
  DCO + GPG validation as a single Go binary with official GHA.
  Alternative: squash merges + `action-semantic-pull-request` to
  lint only PR titles (lightest weight, pairs with release-please).
- **IBM/tekton-lint**: Downgraded from "Consider" to "Skip" — 31
  stars, 2+ year stale, lost original maintainer. Use kubeconform
  with Tekton CRD schemas for structural validation instead.
- **actionlint + zizmor**: Confirmed complementary, not competing.
  actionlint for correctness (syntax, types, shell), zizmor for
  security (injection, permissions, unpinned actions). Run both.

## 3. Security Scanning

| Tool | What It Does | Adopt? | Phase | Audit Notes |
| --- | --- | --- | --- | --- |
| govulncheck | Go vuln scanning (reachability) | Yes | 1 | BSD-3. Official Go team. Gold standard |
| CodeQL | SAST via GitHub Code Scanning | Yes | 1 | MIT action. Free for OSS. Industry standard |
| OSSF Scorecard | Supply chain security scoring | Yes | 1 | Apache-2.0. OpenSSF/Linux Foundation |
| zizmor | GHA security (also in Non-Go) | Yes | 1 | MIT. 5K stars. Complements actionlint |
| SHA-pinned actions check | Verify GHA SHA pins | Yes | 1 | MIT. 50 stars. Partially redundant w/ Scorecard |
| dependency-review-action | Block PRs with vuln deps | Yes | 1 | MIT. GitHub official |
| GODEBUG flags | tar/zip path traversal prevention | Yes | 1 | Go toolchain (BSD-3). Not a tool, just a flag |
| Anchore/grype | Container vuln scanning | Yes | 2 | Apache-2.0. 11.5K stars. Faster than Trivy |
| TruffleHog | Verified secrets scanning | Yes | 2 | **AGPL-3.0**. Fine as standalone CI tool only |
| harden-runner | GHA network egress control | Yes | 2 | Apache-2.0. StepSecurity (SaaS backend) |
| Cosign | Keyless image/artifact signing | Yes | 3 | Apache-2.0. 5.9K stars. OpenSSF/Sigstore |
| Syft | SBOM generation (SPDX/CycloneDX) | Yes | 3 | Apache-2.0. 8.4K stars. Multi-ecosystem |
| SLSA provenance | Build provenance attestations | Yes | 3 | Apache-2.0. OpenSSF. Low stars but backed |
| Trivy | Multi-branch vuln scanning | Yes | 3 | Apache-2.0. 31.7K stars. Mar 2026 compromise |
| OSV-Scanner | Dual-mode vuln scanning | Consider | 2 | Apache-2.0. Google. Wraps govulncheck for Go |

**Audit notes**:

- **TruffleHog**: AGPL-3.0 — safe as standalone CI scanner but
  cannot be imported as a library. Alternative: Gitleaks (MIT).
- **Trivy**: March 2026 supply chain compromise (malicious v0.69.4,
  fake binaries on Docker Hub). DB updates suspended temporarily.
  Running Grype alongside provides defense in depth.
- **OSV-Scanner**: Partially redundant with govulncheck for Go-only
  projects — OSV-Scanner wraps govulncheck internally. Adds value
  for multi-language or container scanning.
- **SHA-pinned actions check**: Partially redundant with Scorecard's
  Pinned-Dependencies check. Faster for CI gating. Keep both.
- **harden-runner**: SaaS backend phones home to StepSecurity.
  Some orgs may object. Community tier free for GitHub-hosted only.
- **Grype vs Trivy**: High overlap on container vulns. Different
  databases catch different things. Running both is intentional
  defense in depth. Grype is faster; Trivy is broader (IaC,
  secrets, K8s).

## 4. Testing

All MIT or Apache-2.0 licensed. No concerns except overcover
(7 stars, single maintainer) and Istio tools (must be copied
from monorepo, not standalone packages).

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| Ginkgo/Gomega | BDD test framework + matchers | Yes | 1 |
| envtest | Local API server for controller tests | Yes | 1 |
| `-shuffle=on` test flag | Catch ordering-dependent tests | Yes | 1 |
| `go mod tidy -diff` | Clean go.mod drift check (Go 1.26+) | Yes | 1 |
| KIND clusters | E2E test infrastructure | Yes | 2 |
| Codecov | Coverage reporting with PR comments | Yes | 2 |
| Go native fuzzing | Webhook validation fuzz tests | Yes | 2 |
| go-ordered-test | Test isolation detector (weekly) | Yes | 2 |
| go-stress-test | Flaky test detector (nightly) | Yes | 2 |
| overcover | Coverage ratchet (only goes up) | Yes | 2 |
| upgrade E2E | N-1 to N version upgrade testing | Yes | 3 |
| system validation | Deployment correctness script | Yes | 3 |
| go-test-split-action | Integration test parallelization | Consider | 3 |
| gotesplit | E2E parallelization via artifacts | Consider | 3 |
| CIFuzz/OSS-Fuzz | Continuous fuzzing (10-20 min/PR) | Consider | 3 |
| version skew testing | N-1 compat between components | Consider | 3 |
| binary size tests | Prevent binary bloat (min/max MB) | Consider | 3 |
| KWOK + k6 | Controller performance testing | Consider | 3 |
| CANNIER | ML-informed flake detection | Consider | 3 |

## 5. CRD and API Validation

All Apache-2.0 licensed. crdify (14 stars) is early-stage.

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

All Apache-2.0 or MIT licensed. ko (8.3K stars) is well-established.

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

All MIT or Apache-2.0. Note: release-please GitHub App was shut
down Aug 2025 — must use the GitHub Action instead.

| Tool | What It Does | Adopt? | Phase |
| --- | --- | --- | --- |
| Dependabot | GHA monthly + Go modules weekly | Yes | 1 |
| Per-PR changelog files | Contour pattern, CI-enforced | Yes | 2 |
| release-please | Automated versioning (Action, not App) | Yes | 3 |
| GoReleaser | Cross-platform binary builds | Consider | 3 |
| Dependabot auto-fix | Regenerate code on dep updates | Yes | 2 |
| Backport action | Auto cherry-pick on label | Yes | 3 |
| GoReleaser dry-run on config change | Catch release config bugs | Yes | 3 |
| Release artifact verification | Diff release vs source | Yes | 3 |
| Fake release smoke test | Daily `v9.9.9-fake` build | Yes | 3 |
| Renovate | Advanced dep management (if needed) | Consider | 3 |

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

4. **KAL with selective checks** — MCN starts fresh but KAL is
   pre-release. Enable jsontags, optionalorrequired, requiredfields,
   defaults, statussubresource, nobools first. Expand as KAL
   stabilizes.

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

**Health audit**:

- Stars: ~18,900. Contributors: 1,571 forks. Active: 20+ releases
  since v2 launch (March 2025).
- License: GPL-3.0-only (formerly AGPL). Fine for dev tool — never
  distributed with Apache-2.0 project. Same as using GCC to build
  proprietary software.
- Bus factor: HIGH RISK — single primary maintainer (Ludovic
  Fernandez). Funded ~$18K/year via Open Collective. No corporate
  backing. No CNCF project status.
- Security: No direct compromise ever. Release attestations and
  SBOMs published. No cosign/sigstore signatures. Linter vetting
  is informal (go/analysis API required, PR review by maintainer).
- Supply chain: 114 bundled linters create broad attack surface.
  No bundled linter has ever been compromised. go/analysis
  framework limits what linter code can execute.
- Coverage: Subsumes all mainstream Go linters. Running any bundled
  linter standalone is fully redundant.
- Operational: Memory-hungry (staticcheck can use 34GB+). Set
  `GOGC=50`, `--timeout=10m`, reduce `--concurrency` if needed.
- Alternatives: None viable at scale. De facto standard.
- Risk mitigation: Pin versions, pin GHA by SHA, monitor project
  health, be prepared to fork if maintainer steps away.

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

**Health audit**:

- Stars: 136. Contributors: 25. Commits: ~388. Created Dec 2024.
  Active: weekly merges, last push May 18, 2026.
- Governance: Official SIG API Machinery subproject. OWNERS:
  JoelSpeed (Red Hat), jpbetz (Google, SIG lead), sivchari,
  everettraven. Biweekly dedicated meeting.
- License: Apache-2.0. Fully compatible with Apache-2.0 projects.
  No license concerns whatsoever.
- Security: Pure static analysis (go/analysis AST inspection). No
  CVEs. Does not execute analyzed code, no network calls. Falls
  under standard Kubernetes security disclosure process.
- Coverage: 31 rules, ALL unique. Zero overlap with any built-in
  golangci-lint linter. Fills gap that nothing else provides.
- Maturity: Pre-release. ZERO tagged releases. No semver, no
  changelog. Any pseudo-version bump could contain breaking
  changes. Pin explicitly and test after updates.
- Adoption: ~20 repos including cluster-api, openshift/api,
  openshift/hypershift, cert-manager/trust-manager, Kong,
  metal3-io, kgateway.
- Custom binary risk: golangci-lint custom builds are NOT
  reproducible (issue #5961). Cache by config hash + Go version.
  Build in constrained CI step (no secrets). Larger attack
  surface than standard golangci-lint. Mitigate by pinning both
  golangci-lint version and KAL pseudo-version.

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

**Health audit**:

- Stars: ~30. Single maintainer (Denis Tingaikin, Network Service
  Mesh/CNCF contributor). Burst-style releases: v0.5.0 (Feb 2024),
  v1.0.0 (Aug 2025).
- License: GPL-3.0. Fine for dev tooling — not linked into shipped
  binary. golangci-lint itself is also GPL-3.0.
- golangci-lint pins go-header at v0.5.0. Insulated from upstream
  churn. If upstream dies, golangci-lint can fork or deprecate.
- Coverage: Only tool with native golangci-lint integration for
  license headers. Unique template/regex/YEAR-RANGE system. Does
  NOT overlap with addlicense or SkyWalking Eyes (those fix,
  goheader checks).
- Known bugs: Most recent false-positive (#5284) fixed Jan 2025.
  `only-new-issues` interaction (#2470) is a known limitation.
- Recommendation: goheader for Go enforcement + google/addlicense
  for bulk insertion and non-Go files.

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

**Health audit**: 17 stars, Apache-2.0 license. Single maintainer
(Julian Friedman). Latest release v0.2.0 (Dec 2024) fixed the
non-determinism bug. Unique in golangci-lint — no other linter
enforces import alias consistency. Vendored in golangci-lint,
insulated from upstream tempo. Clean.

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

**Health audit**: Part of golang.org/x/tools. BSD-3-Clause. Official
Go team (Alan Donovan). Continuously updated. Known panics in
stringscut (#77451) and rangeint (#77161) — treated as release
blockers by Go team. Partial overlap with gocritic style checks.
Vendored in golangci-lint, pinned version insulates from panics.

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

**Health audit**: 18 stars, Apache-2.0, solo maintainer. v0.6.0
(April 2026). New linter, low adoption. Simple AST checks — low
bug surface. Unique in golangci-lint. Vendored.

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

**Health audit**: 10 stars, MIT, solo maintainer. v0.3.0 (May
2026). Competitor smrcptr exists but lacks golangci-lint
integration. Kyma project (K8s) adopted recvcheck successfully
with exclusions for DeepCopyObject and String.

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

**Health audit**: 18 stars, Apache-2.0, solo maintainer. v1.4.3
(May 2026, active). Zero open issues. `identical` is safe;
`opaque`/`unused` confirmed noisy for K8s operator patterns.

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

**Health audit**: ~200 stars, GPL-3.0 (fine for tooling), small
team (3 contributors). v2 stable. Last published March 2025 —
low activity but functional. Complements gomodguard (package-level
vs module-level). Confirmed no overlap.

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

**Health audit**: 217 stars, MPL-2.0, go-simpler org. v0.12.0
(recent). Active development. Listed on Go Wiki slog resources.
Only useful if using slog — irrelevant for logr+zap.

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

**Health audit**: 91 stars, Apache-2.0, solo maintainer (yeya24).
v0.3.0 (April 2024 — over 1 year old). Low recent activity.
Wraps upstream prometheus/client_golang promlint. If promlinter
is abandoned, fall back to `testutil.CollectAndLint()` in tests.

### exhaustive

**Current version**: 339 stars, 45 releases. Last release Nov 2023
(18+ month gap).

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

**Health audit**: 339 stars, BSD-2-Clause, solo maintainer. Last
release v0.12.0 (Nov 2023 — 18+ month gap). Memory issues with
golangci-lint (#5065). Proto-generated enum noise requires careful
config. Use `explicit-exhaustive-switch` mode for conservative
adoption. Unique coverage — no other linter does this.

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
hygiene. **Note**: Redundant with gosec G116 — if gosec is enabled
(which it is via golangci-lint), consider dropping bidichk to avoid
duplicate checks.

**Health audit**: 42 stars, MIT, solo maintainer. v0.3.3 (March
2025). 0 open issues. Simple byte scanner — minimal bug surface.
Near-complete overlap with gosec G116 rule. Keep only if you want
granular per-rune configuration that gosec doesn't offer.

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

**Health audit**: 244 stars, BSD-3-Clause, maintained by Fatih
Arslan (vim-go creator). v1.15.0 (March 2025). Not in golangci-lint
(issue #1025 remains open). depguard + forbidigo confirmed to fully
cover faillint's features with better integration. SKIP confirmed.

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

**Health audit**: 167 stars, Apache-2.0, solo maintainer (Andrew
Brown). Latest release v2.3.1 (April 2026, very recent). Fully
subsumes faillint when paired with depguard. `analyze_types` mode
resolves through aliases/embedding. Clean.

## Health Audit Summary — All 15 Go Tools

| Tool | Stars | License | Maintainer | Risk | Unique? |
| --- | --- | --- | --- | --- | --- |
| golangci-lint v2 | 18.9K | GPL-3.0 | Solo (ldez) | Bus factor | Yes |
| KAL | 136 | Apache-2.0 | SIG API Mach | Pre-release | Yes (31 rules) |
| goheader | 30 | GPL-3.0 | Solo | Vendored | Unique in GCL |
| importas | 17 | Apache-2.0 | Solo | Vendored | Unique in GCL |
| modernize | N/A | BSD-3 | Go team | Low | Partial gocritic |
| funcorder | 18 | Apache-2.0 | Solo | New/small | Unique in GCL |
| recvcheck | 10 | MIT | Solo | Small | vs smrcptr |
| iface | 18 | Apache-2.0 | Solo | FP risk | Unique in GCL |
| depguard | 200 | GPL-3.0 | Small team | Stable | vs gomodguard |
| sloglint | 217 | MPL-2.0 | go-simpler | Active | slog-only |
| promlinter | 91 | Apache-2.0 | Solo | Stale (1yr) | vs test promlint |
| exhaustive | 339 | BSD-2 | Solo | 18mo gap | Unique |
| bidichk | 42 | MIT | Solo | Redundant | vs gosec G116 |
| faillint | 244 | BSD-3 | Fatih Arslan | SKIP | Covered by above |
| forbidigo | 167 | Apache-2.0 | Solo | Active | Function-level |

**License summary**: All fine for dev tooling. GPL-3.0 (golangci-lint,
goheader, depguard) is standard for Go linting tools — never
distributed with your project.

**Updated recommendations based on audits**:

- **bidichk**: Dropped — redundant with gosec G116 (already
  enabled via golangci-lint).
- **exhaustive**: Add "configure carefully" warning — 18-month
  release gap, memory issues, proto noise. Use
  `explicit-exhaustive-switch` mode.
- **promlinter**: Add note that upstream `testutil.CollectAndLint()`
  provides runtime-equivalent coverage as a safety net.
- **faillint**: SKIP confirmed — depguard + forbidigo fully covers
  its features with golangci-lint integration.

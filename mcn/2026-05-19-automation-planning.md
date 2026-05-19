---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [automation, ci, cd, release, linting, testing, konflux, claude-code]
---

# MCN Automation Planning — Submariner Automation Review

Comprehensive review of all Submariner automation, organized by category.
Each section covers what Submariner does, whether MCN should adopt it,
and any notes on adaptation.

## 1. Build System

### 1.1 Makefile Infrastructure (Shipyard)

Submariner uses a shared Makefile system distributed via the "shipyard"
repo. All component repos include `Makefile.inc` which provides
standardized targets. Shipyard has 8 Makefile fragments:

- `Makefile` — top-level orchestrator (deploy, e2e, clusters, clean)
- `Makefile.linting` — all lint targets
- `Makefile.images` — container image build/push/preload
- `Makefile.clusters` — KIND/OCP/ACM cluster management
- `Makefile.dapper` — containerized build environment
- `Makefile.versions` — version calculation
- `Makefile.shipyard` / `Makefile.inc` — shared includes and `using=`
  flag system

Key targets across repos: `build`, `unit`, `e2e`, `images`,
`multiarch-images`, `release-images`, `golangci-lint`, `markdownlint`,
`shellcheck`, `yamllint`, `gitlint`, `govulncheck`, `deploy`,
`clusters`, `clean`, `post-mortem`, `backport`.

**MCN recommendation**: Adopt a simplified single-repo Makefile with
standard targets (build, test, lint, images, deploy, clean). No need
for the multi-repo shipyard include system — MCN can define everything
in one Makefile. The `using=` flag composition system is clever but
overkill for a single project.

### 1.2 Dapper (Containerized Build Environment)

Submariner wraps all builds in a "Dapper" container — a Fedora-based
image with Go, Docker, KIND, kubectl, Helm, golangci-lint, govulncheck,
gh CLI, skopeo, UPX, and all shared scripts baked in. This ensures
reproducible builds across developer machines and CI.

**MCN recommendation**: Consider but don't adopt initially. Modern CI
runners (GitHub Actions with pinned tool versions) provide sufficient
reproducibility. Dapper adds complexity (Docker-in-Docker, SELinux
context, socket mounting) that a new project doesn't need. Revisit if
reproducibility becomes a problem.

### 1.3 Go Compilation

Submariner uses LDFLAGS to embed version info at compile time:
`-X pkg/version.Version=$(VERSION)`. Cross-compilation is supported
via GOARCH/GOOS with architecture mapping for Docker buildx platforms.
UPX compression is optionally applied to shrink binaries.

**MCN recommendation**: Adopt LDFLAGS version embedding. Skip UPX
compression unless image size is a concern.

### 1.4 Code Generation

Submariner-operator uses:

- `controller-gen object` — generates DeepCopy methods from Go types
- `controller-gen crd` — generates CRD YAMLs from Go type annotations
- `client-gen` / `kube_codegen.sh` — Kubernetes API client generation
- `protoc` — protobuf compilation (submariner core only)
- `operator-sdk generate bundle` — OLM bundle manifest generation
- `gen-codeowners` — CODEOWNERS from template

CI verifies generated code matches committed code (diff check after
regeneration).

**MCN recommendation**: Adopt controller-gen for CRDs and DeepCopy.
Add a CI check that regenerates and diffs to catch stale generated
code. Skip protobuf unless MCN needs it. OLM bundle generation needed
if shipping via OLM.

## 2. Container Images

### 2.1 Upstream Dockerfiles

Multi-stage builds with 3 stages:

1. Builder (shipyard-dapper-base) — compiles Go binary
2. Base (Fedora) — installs system packages via `dnf_install`
3. Final (scratch) — minimal image with just binary + libs

Multi-arch via `BUILDPLATFORM`/`TARGETPLATFORM` build args
(amd64, arm64). Non-root user (UID 1001).

**MCN recommendation**: Adopt multi-stage scratch-based Dockerfiles.
Multi-arch from day one if required.

### 2.2 Downstream Konflux Dockerfiles

Separate `*.konflux` Dockerfiles for Red Hat builds:

- Base images: `registry.redhat.io/ubi9/go-toolset:latest` (build),
  `registry.redhat.io/ubi9/ubi-minimal:latest` (runtime)
- FIPS compliance: `GOEXPERIMENT=strictfipsruntime`
- Version labels: `version`, `release`, `cpe` baked per branch
- Red Hat image names injected via linker flags
- `SOURCE_DATE_EPOCH` for reproducible builds
- 4 architectures: x86_64, arm64, ppc64le, s390x

**MCN recommendation**: Will need Konflux Dockerfiles once downstream
builds are set up. Follow the same UBI9 base image pattern. FIPS
compliance is mandatory for OpenShift operators.

### 2.3 Image Publishing

Upstream: images pushed to `quay.io/submariner/*` on every merge to
devel/release branches via GitHub Actions. Uses skopeo for multi-arch
manifest push.

For releases, images are **re-tagged** (not rebuilt) using
`skopeo copy --all` — promotes tested images without introducing
untested changes.

**MCN recommendation**: Adopt image push on merge to main/release
branches. Adopt the re-tag-not-rebuild pattern for releases.

### 2.4 RPM Lockfiles

Components needing system RPMs (libreswan, iptables, etc.) use
lockfiles at `.rpm-lockfiles/<component>/rpms.lock.yaml` to pin
exact RPM versions across all 4 architectures. Update script uses
podman with Red Hat entitlements.

**MCN recommendation**: Only needed if MCN containers install RPMs
beyond what UBI provides. If so, adopt the lockfile pattern for
reproducible builds.

## 3. Linting and Code Quality

### 3.1 Go Linting (golangci-lint)

Submariner uses golangci-lint v2 with 60+ linters enabled. Key ones:

- `govet` (with fieldalignment), `gosec`, `gocyclo` (max 15)
- `lll` (140 char line length), `goheader` (Apache-2.0 license)
- `wrapcheck`, `err113`, `ginkgolinter`, `modernize`
- Formatters: `gci`, `gofmt`, `gofumpt`, `goimports`
- Relaxed rules for test files
- Generated code excluded

**MCN recommendation**: Adopt. Start with a strict config and relax
as needed. The Submariner config is a good starting point — copy
`.golangci.yml` and adjust.

### 3.2 Other Linters

- `markdownlint` (`.markdownlint.yml`) — 140 char lines
- `yamllint` (`.yamllint.yml`) — 140 char lines, ignores generated
  dirs
- `shellcheck` (`.shellcheckrc`) — shell script linting
- `gitlint` (`.gitlint`) — commit message format enforcement
- `staticcheck` (`staticcheck.conf`) — custom initialisms
- `markdown-link-check` — validates links in markdown files
- `packagedoc-lint` — Go package documentation
- `lichen` — dependency license audit (CNCF-approved licenses)

**MCN recommendation**: Adopt all of these. They're low-effort to set
up and catch real issues. Start with markdownlint, yamllint,
shellcheck, gitlint as day-one linters. Add lichen for license
compliance.

### 3.3 Security Scanning

- `govulncheck` — Go vulnerability scanning, generates SARIF for
  GitHub Security tab
- `grype` — container image vulnerability scanning with false
  positive suppression (`.grype.yaml`)
- `CodeQL` — variant analysis on PRs
- `Anchore` — vulnerability scan with SARIF upload, fail on HIGH
  (PR) or report-only (merge)

**MCN recommendation**: Adopt govulncheck and CodeQL from day one.
Add Anchore/grype once container images are being built.

## 4. Testing

### 4.1 Unit Tests

Ginkgo/Gomega test framework with JUnit XML and coverage output.
Coverage reported to SonarQube. Tests run on every PR.

**MCN recommendation**: Adopt Ginkgo/Gomega (Kubernetes ecosystem
standard) or standard Go testing. JUnit XML output for CI integration.
Coverage reporting from the start.

### 4.2 E2E Tests

Shipyard provides a full e2e framework (17 Go files) for multi-cluster
testing: cluster management, namespace lifecycle, network pod creation,
connectivity verification, service export/import, globalnet/FIPS
detection.

Two-tier CI execution:

- **Default E2E**: Runs on every PR (basic connectivity)
- **Full E2E matrix**: Only on `ready-to-test` label (cable drivers x
  CNI x globalnet x K8s versions — 20+ combinations)

**MCN recommendation**: Build a lightweight e2e framework specific to
MCN's use case. Adopt the label-gated matrix pattern — run basic e2e
on every PR, full matrix only when ready. The `ready-to-test` label
after 2 approvals is a good gating pattern.

### 4.3 Upgrade Tests

Deploys latest released version, upgrades to PR version, validates
functionality. Runs on PRs targeting devel.

**MCN recommendation**: Adopt once there's a first release to upgrade
from. Critical for operator projects.

### 4.4 Flake Finder

Twice-daily cron runs the full E2E matrix on devel branch to surface
flaky tests without blocking PRs. Creates issues automatically.

**MCN recommendation**: Adopt once E2E tests exist. Run weekly or
daily depending on test suite size.

### 4.5 System Tests

475-line bash script that exhaustively validates deployed state:
namespaces, CRDs, service accounts, roles, deployments, DaemonSets,
pod status, env vars, security contexts, volume mounts, secrets.

**MCN recommendation**: Adopt the concept — a deployment validation
script that verifies everything is correctly deployed. Very valuable
for operator projects.

## 5. CI/CD Workflows (GitHub Actions)

### 5.1 PR Workflows

Submariner-operator has 18 workflow files. Key PR workflows:

- `linting.yml` — 15 linting jobs including bundle validation, CRD
  freshness, CodeQL, Anchore, all the text linters
- `unit.yml` — unit tests with artifact collection
- `e2e.yml` — default E2E on non-draft PRs
- `e2e-full.yml` — full matrix on `ready-to-test` label
- `multiarch.yml` — multi-arch image build verification
- `system.yml` — deployment validation
- `upgrade-e2e.yml` — upgrade testing
- `branch.yml` — enforces PRs target devel/main
- `codeowners.yml` — CODEOWNERS regeneration check
- `subctl-unit.yml` — consuming project compatibility

**MCN recommendation**: Start with: linting, unit tests, basic e2e,
branch enforcement. Add multiarch, upgrade, and full e2e matrix as
the project matures.

### 5.2 Merge/Release Workflows

- `release.yml` — builds and pushes images on merge to
  devel/release-* branches
- `report.yml` — coverage to SonarQube, CodeQL, Anchore on merge

**MCN recommendation**: Adopt image push on merge from day one.
Coverage reporting is nice-to-have initially.

### 5.3 Periodic Workflows

- `periodic.yml` — weekly full markdown link check, creates issues
  for broken links
- `flake_finder.yml` — twice-daily E2E flake detection
- `stale.yml` — daily stale issue/PR cleanup (120d issues, 14d PRs)
- `dependent-issues.yml` — cross-issue dependency tracking

**MCN recommendation**: Adopt stale.yml and periodic link checking
from day one. Add flake finder when E2E exists.

### 5.4 Bot Automation

`.submarinerbot.yaml` auto-labels PRs `ready-to-test` after 2
approvals. This gates expensive E2E runs behind review.

**MCN recommendation**: Adopt this pattern to save CI resources.

## 6. Release Automation

### 6.1 Upstream Release (Multi-Repo State Machine)

Submariner uses a 6-stage state machine across 8 repos:
`branch -> shipyard -> admiral -> projects -> installers -> released`

Declarative YAML files in a `releases/` repo drive the process.
Merging a YAML file triggers the release workflow. Stages create
branches, tag repos, update cross-repo dependencies, and promote
images — all in strict dependency order.

Version types: milestone (m0), release candidate (rc0), GA, patch.

**MCN recommendation**: Do NOT adopt the multi-repo state machine —
it's designed for 8 interconnected repos. MCN should use a simpler
approach: tag-triggered release workflow. However, the declarative
YAML concept and dry-run validation on PRs are worth adopting.

### 6.2 Downstream Release (Konflux 20-Step Workflow)

Submariner's downstream release is a 20-step process orchestrated
through Claude Code skills and shell scripts. Steps 1-7 are setup,
8-14 are stage release, 15-20 mirror the stage flow for prod:

1. Create upstream release branch (Y-stream only)
2. Configure Konflux downstream (tenant, RPAs, overlays)
3. Set up Tekton pipelines per component + bundle
4. Fix Enterprise Contract violations
5. CVE scanning + version label updates
6. Cut upstream release tags
7. Update bundle image SHAs from Konflux snapshots
8. Create component stage Release CR
9. Add release notes from Jira
10. Apply stage release, verify builds
11. Update FBC catalog with new bundle
12. Create FBC stage releases (one per OCP version)
13. Apply FBC stage releases, verify builds
14. Share stage FBC URLs with QE
15. Create component prod Release CR
16. Apply prod release, verify builds
17. Create FBC prod releases
18. Apply FBC prod releases, verify builds
19. Share prod FBC URLs with QE
20. Update FBC templates with prod URLs

**MCN recommendation**: This process will be needed once MCN ships
via Konflux. Start by understanding the concepts (Release CRs,
snapshots, FBC catalogs, stage vs prod). The Claude Code skills that
automate these steps can be adapted for MCN.

### 6.3 FBC (File-Based Catalog) Management

OLM catalogs built per OCP version (4.16-4.21). A master
`catalog-template.yaml` defines channels/bundles, rendered into
per-version catalogs. Tekton pipelines build catalog images.

Image mirror sets map `registry.redhat.io` to `quay.io` staging
URLs for pre-release testing. Auto-conversion to prod URLs on
release.

**MCN recommendation**: Needed if MCN ships as an OLM operator
(likely). Can follow the same catalog-template pattern.

### 6.4 Image Promotion and Signing

- Stage: images pushed to `registry.stage.redhat.io/rhacm2/`
- Prod: images pushed to `registry.redhat.io/rhacm2/` with
  `hacbs-signing-pipeline-config-redhatrelease2` signing
- Source containers built alongside (`build-source-image: true`)
- SBOM webhook notifications to Bombino

**MCN recommendation**: Standard Red Hat downstream requirements.
MCN will need all of this once onboarded to Konflux.

## 7. Dependency Management

### 7.1 Dependabot

Configured per branch (devel + each release branch):

- GitHub Actions: monthly updates, grouped
- Go modules: weekly, with ignore lists for cross-repo-managed deps
- Per-branch K8s version pinning (e.g., release-0.22 tracks k8s 0.34)
- Tools directory has separate Go module config

**MCN recommendation**: Adopt from day one. Configure for both
GitHub Actions and Go modules. Much simpler for a single repo —
no cross-repo ignore lists needed.

### 7.2 Renovate / MintMaker

Renovate configured only for Tekton pipeline task updates.
Konflux's MintMaker auto-creates PRs for Tekton task SHA updates,
RPM signature scan updates, and Go dependency digests.

**MCN recommendation**: Renovate for Tekton comes automatically with
Konflux onboarding. No manual setup needed.

### 7.3 Cross-Repo Dependency Integration

Weekly periodic workflow auto-creates PRs that update all
`submariner-io/*` Go dependencies to latest devel versions. Catches
integration issues before release.

**MCN recommendation**: Not needed for a single repo. Useful later
if MCN splits into multiple repos with shared libraries.

## 8. Claude Code / AI Automation

### 8.1 CLAUDE.md Convention

Every Submariner repo has a CLAUDE.md with:

- Development instructions (commit standards, testing commands)
- Links to workflow documents in `.agents/workflows/`
- Project-specific build/test guidance

**MCN recommendation**: Adopt from day one. CLAUDE.md should document
build commands, test procedures, commit conventions, and link to
any agentic workflows.

### 8.2 Agentic Workflows (.agents/workflows/)

Structured markdown documents that guide Claude Code through
multi-step operational tasks. ~35 unique workflow documents across
repos covering:

- CVE fix lifecycle (scanning, updating, verifying, committing)
- Konflux component/bundle setup on new branches
- Konflux CI diagnosis and Enterprise Contract fixes
- Bundle SHA updates from Konflux snapshots
- Commit message templates

**MCN recommendation**: Start with a CVE fix workflow and commit
templates. Add Konflux workflows as downstream builds are set up.
This is high-value, low-effort automation.

### 8.3 Claude Code Skills (Plugins)

Submariner uses 4 plugin marketplaces providing 40+ skills:

**shipyard@submariner** (1 skill):

- `/cve-fix` — full CVE remediation pipeline for Go repos

**release-management@submariner-release** (15 skills):

- Release setup: `/configure-downstream`,
  `/konflux-component-setup`, `/konflux-bundle-setup`
- Build management: `/konflux-ci-fix`, `/rpm-lockfile-update`,
  `/update-version-labels`, `/bundle-image-update`
- Release execution: `/create-component-release`,
  `/create-fbc-release`, `/add-release-notes`, `/get-fbc-urls`
- Operations: `/fbc-update`, `/add-team-member`
- Knowledge: `/learn-release`, `/release-ls`

**jira@ai-helpers** (20+ skills):

- Issue CRUD, triage, status reporting, cross-tool reconciliation
- Project-agnostic, available to any OpenShift team

**claude-skills@claude-skills** (3 skills):

- `/work-summary`, `/jira`, `/notes`

**MCN recommendation**: The Jira skills and personal utility skills
work as-is. CVE fix skill is directly reusable. Release management
skills will need MCN-specific versions once downstream builds exist.

### 8.4 Project Memory

Feedback-driven memory captures team preferences:

- "Always commit" — don't ask, just commit
- "No PR replies without approval" — draft comments first
- "Never change git remote URLs" — user uses SSH + YubiKey
- Project-specific context (who does what, which versions exist)

**MCN recommendation**: Build up organically as the project develops.
No upfront setup needed.

## 9. Repository Governance

### 9.1 Branch Protection

- All PRs must target devel/main (enforced by `branch.yml`)
- `ready-to-test` label gates expensive CI (auto-applied after 2
  approvals)
- Stale issues closed after 120 days, PRs after 14 days
- `confirmed` and `security` labels exempt from stale cleanup

**MCN recommendation**: Adopt all of these from day one.

### 9.2 CODEOWNERS

Generated programmatically from `CODEOWNERS.in` template using a
Python script. CI verifies the generated file matches the template.

**MCN recommendation**: Nice-to-have. For a small team, a hand-written
CODEOWNERS is fine initially.

### 9.3 Issue/PR Templates

Release tracker template with pre-release checklist. Dependent issues
workflow tracks cross-PR dependencies.

**MCN recommendation**: Add issue templates for bugs and features.
Dependent issues workflow is useful for coordinated changes.

## 10. Automation NOT Recommended for MCN

### 10.1 Dapper (Containerized Builds)

Heavy Docker-in-Docker setup with custom entrypoint, SELinux handling,
socket mounting. Adds significant complexity for marginal
reproducibility gains over GitHub Actions runners.

**Why not**: Complexity cost outweighs benefit for a new project.
Revisit if build reproducibility becomes a problem.

### 10.2 Multi-Repo Release State Machine

6-stage release across 8 repos with strict dependency ordering.
Elegant but designed for a very specific multi-repo architecture.

**Why not**: MCN is a single operator project (or at most 2-3 repos).
A simple tag-triggered release is sufficient.

### 10.3 Shipyard Shared Makefile Includes

Cross-repo Makefile distribution via `Makefile.inc` downloads.
Powerful for consistency across 8 repos but adds indirection.

**Why not**: Single repo doesn't need shared includes. Define
everything directly.

### 10.4 Scale Testing (10 Clusters)

KIND-based 10-cluster scale testing with kernel parameter tuning.

**Why not**: Premature for a new project. Add when scale becomes
a concern.

### 10.5 Krew Plugin Distribution

kubectl plugin distribution via krew index updates.

**Why not**: Only needed if MCN has a kubectl plugin CLI.

## 11. Suggested Adoption Phases

### Phase 1 — Day One (Before Any Code)

- Makefile with standard targets (build, test, lint, images)
- golangci-lint config (copy from Submariner, adjust)
- markdownlint, yamllint, shellcheck, gitlint configs
- GitHub Actions: linting, unit tests, branch protection
- Dependabot for GitHub Actions and Go modules
- CLAUDE.md with project conventions
- `.agents/commit-templates.md`
- Stale issue/PR workflow
- LICENSE, README, CONTRIBUTING, .gitignore

### Phase 2 — First Code Landing

- controller-gen for CRDs and DeepCopy
- CI check for stale generated code
- govulncheck in CI
- CodeQL variant analysis
- Container image builds (multi-stage, scratch-based)
- Image push on merge to main
- Basic e2e test framework
- `.agents/workflows/cve-fix.md`

### Phase 3 — Approaching First Release

- Release workflow (tag-triggered or YAML-driven)
- Upgrade e2e testing
- Full e2e matrix with label gating
- Multi-arch image builds
- License checking (lichen)
- SonarQube coverage reporting
- Flake finder (periodic e2e)
- System validation script

### Phase 4 — Downstream / Konflux Onboarding

- Konflux Dockerfiles (UBI9, FIPS, multi-arch)
- Tekton pipeline configs
- Tenant configuration in konflux-release-data
- Enterprise Contract policy
- RPM lockfiles (if needed)
- OLM bundle generation
- FBC catalog setup
- Release management skills for Claude Code
- Stage/prod release workflow
- Container signing and SBOM

## 12. CI Scaffold Plan — Day One Files

This section is the concrete implementation of Phase 1 from section
11. These files go into the MCN repo root before any code lands.
22 files total.

### 12.1 File Inventory

```text
.golangci.yml               — Go linting (adapted from submariner-operator)
.markdownlint.yml            — Markdown linting (140 char lines)
.yamllint.yml                — YAML linting (140 char lines)
.shellcheckrc                — Shell script linting (SC1090, SC2154)
.gitlint                     — Commit message linting
.grype.yaml                  — Vuln scan false positive suppression
.markdownlinkcheck.json      — Markdown link check config
staticcheck.conf             — Go staticcheck initialisms
.github/dependabot.yml       — GHA monthly + Go mod weekly updates
.github/workflows/
  linting.yml                — 12-job linting (PR trigger)
  unit.yml                   — Go unit tests (PR trigger)
  branch.yml                 — Enforce PR targets main/release-*
  stale.yml                  — Close stale issues/PRs (daily cron)
  periodic.yml               — Weekly full link check (Sunday cron)
  release.yml                — Build/push images (merge trigger)
  codeowners.yml             — Verify CODEOWNERS (PR trigger)
  ai-security-review.yml     — Claude security review (PR trigger)
  ai-rbac-review.yml         — Claude RBAC review (path-filtered PR)
  ai-release-notes.yml       — Claude release note suggestions (PR)
Makefile                     — Standalone build system (no Shipyard)
Dockerfile                   — Multi-stage operator image
CLAUDE.md                    — Claude Code project conventions
```

### 12.2 Linting Configs — Adaptation Notes

**`.golangci.yml`** — Copy from `submariner-operator/.golangci.yml`
(248 lines, v2 format) with these changes:

- goheader template: "Submariner project" to "MCN project"
- Remove exclusions for `pkg/embeddedyamls/yamls.go` (Submariner
  generated file)
- Remove exclusion for `BrokerK8sApiServer` struct field
- Remove exclusion for `pkg/metrics/service-monitor.go` goheader
- Keep everything else: 60+ linters, 140 char lines,
  Ginkgo/Gomega dot-imports, all 4 formatters (gci, gofmt,
  gofumpt, goimports), govet fieldalignment, gocyclo min 15

**Other configs** — Direct copies from Submariner with minimal
MCN-specific path adjustments in `.yamllint.yml` ignore list.

### 12.3 Linting Workflow — 12 Parallel Jobs

Adapted from Submariner's 15-job `linting.yml`. Uses direct tool
invocation instead of Shipyard `make` targets. Go-specific jobs
guarded with `if: hashFiles('go.mod') != ''` to skip gracefully
before Go code exists.

Jobs:

1. `apply-suggestions-commits` — block "Apply suggestions" and
   "fixup!" commits (same SHA-pinned actions as Submariner)
2. `gitlint` — commit message format via `pip install gitlint`
3. `golangci-lint` — via `golangci/golangci-lint-action`
4. `markdown-link-check` — modified files only via
   `gaurav-nelson/github-action-markdown-link-check`
5. `markdownlint` — via `npx markdownlint-cli2`
6. `shellcheck` — `find . -name "*.sh" | xargs shellcheck`
7. `yamllint` — via `pip install yamllint`
8. `govulncheck` — Go vulnerability scanning
9. `variant-analysis` — CodeQL for Go with SARIF
10. `vulnerability-scan` — Anchore with SARIF upload, fail on HIGH
11. `crds` — verify generated CRDs match committed (controller-gen
    diff check)
12. `licenses` — dependency license audit (lichen)

Dropped from Submariner: `bundle` (no OLM bundle yet),
`check-branch-dependencies` (single repo), `yamls` (no embedded
YAMLs), `packagedoc-lint` (add later).

### 12.4 AI Agent PR Reviews — New for MCN

All three use `anthropics/claude-code-action@v1` with
`ANTHROPIC_API_KEY` repo secret. All non-blocking (informational
PR comments only). All use `use_sticky_comment: true` to update
the same comment on new pushes rather than flooding.

#### Security Review Agent

- Trigger: every non-draft PR
- Analyzes diff for: RBAC changes (ClusterRole, Role,
  RoleBinding), privilege escalation, secrets handling, security
  context changes (runAsRoot, privileged, capabilities), network
  policy changes, hardcoded credentials, undigested image refs
- Confidence scoring, only reports findings with confidence >= 80
- Severity levels: CRITICAL, HIGH, MEDIUM
- Read-only tools: `gh pr diff`, `grep`, `find`, `Read`

#### RBAC Change Review Agent

- Trigger: PRs that modify RBAC-related files only
- Path filters: `config/rbac/**`, `**/role*.yaml`,
  `**/clusterrole*.yaml`, `**/*_types.go`, `**/controller*.go`
- Detailed permission change analysis against controller source
- Least-privilege verification — does the controller actually use
  these permissions?
- Output: summary table (Resource, Verbs, Scope, Justification)
- Flags wildcard verbs/resources and cluster-scope that could be
  namespace-scope

#### Release Notes Agent

- Trigger: every non-draft PR
- Detects: API/CRD field changes, breaking changes, new features,
  deprecations, config changes, default value changes, new
  controller behaviors
- Internal-only changes (refactoring, tests, CI, dep bumps) get
  "No release note needed" response
- Suggests actual release note text with category: Feature, Bug
  Fix, Breaking Change, Deprecation, Enhancement
- Checks for existing release note labels/annotations

### 12.5 Supporting Files

**Makefile** — Standalone (no Shipyard includes). Targets: build,
test, lint, golangci-lint, markdownlint, yamllint, shellcheck,
govulncheck, manifests, generate, images, clean, help. Version
from git tags via LDFLAGS.

**Dockerfile** — Multi-stage: `golang:1.24` builder with
TARGETARCH cross-compilation, `distroless/static:nonroot` final.

**CLAUDE.md** — Project conventions: build commands, commit
standards (--signoff required), test framework (Ginkgo/Gomega),
license (Apache-2.0), line length (140 chars).

### 12.6 Secrets Required

- `ANTHROPIC_API_KEY` — Claude API access (AI review workflows)
- `QUAY_USERNAME` / `QUAY_PASSWORD` — push images (release.yml)

### 12.7 CI Platform Split

- **GitHub Actions**: all linting, unit tests, security scanning,
  AI reviews, image builds, governance (stale, branch protection)
- **Prow (OpenShift CI)**: e2e and integration tests — separate
  config in `openshift/release` repo, tracked by CORENET-7083

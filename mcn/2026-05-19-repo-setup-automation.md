---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [repo-setup, automation, ci, openshift, multicluster-networking, jira]
---

# New Repo Setup and Automation for OpenShift Multicluster Networking

Planning notes for bootstrapping the OpenShift Multicluster Networking (MCN)
project repositories. Goal: get automation and infrastructure in place before
any application code lands.

## Jira Issues — Full List Under CORENET-7067

Parent epic: **CORENET-7067** — "Create new repos for MCN in upstream and
downstream" (In Progress, Major priority, assigned to Yossi Boaron)

16 child stories total. 1 closed, 15 open. Grouped by area below.

### Upstream Repo Creation and Design

- **CORENET-7076** — Demo MCN to OVNK upstream community — **Closed**
  (Yossi Boaron)
- **CORENET-7077** — Upstream design proposal for MCN
  repository — To Do (Yossi Boaron)
  - Write and submit upstream enhancement proposal
  - Document repository scope and architecture
  - Get community feedback and approval
- **CORENET-7080** — Initial repository setup for MCN
  upstream — To Do (unassigned)
  - Create upstream GitHub repository
  - Set up directory structure, initialize Go modules
  - Add LICENSE, README.md, CONTRIBUTING.md
  - Configure .gitignore, .gitattributes

### Build System (upstream)

- **CORENET-7081** — Add Makefiles and build system to upstream MCN
  repository — To Do (Daniel Farrell)
  - Makefile with standard targets
  - Container builds
  - Build tags and versioning strategy
  - Code generation tools (controller-gen, client-gen)
- **CORENET-7078** — Setup infra for MCN container image registry and
  publishing (upstream) — To Do (Daniel Farrell)
  - Design container image structure
  - Build config, registry location, tagging strategy
  - Configure multiarch builds

### CI/CD (upstream)

- **CORENET-7083** — Prow job configuration for MCN
  repo — To Do (Prachi Yadav)
  - Pre-submit jobs (pre-merge testing)
  - Post-submit jobs (post-merge builds)
  - Periodic prow jobs
  - Add job configuration to OpenShift CI
- **CORENET-7086** — Enable pre-merge testing
  automation — To Do (Daniel Farrell)
  - Unit test execution in pre-merge
  - Code coverage reporting
  - Linting and formatting (golangci-lint, gofmt)
  - License header verification
  - API compatibility checks
- **CORENET-7087** — Enable container image builds in
  CI — To Do (Daniel Farrell)
  - Integrate container builds into Prow jobs
  - Configure image pushing on merge to main
  - Set up multi-arch builds in CI
  - Add image vulnerability scanning

### Downstream

- **CORENET-7082** — Initial repository setup downstream for
  MCN — To Do (Daniel Farrell)
  - Create openshift/mcn repository
  - Configure downstream build system
  - Set up sync automation from upstream
  - Document downstream contribution workflow
- **CORENET-7079** — Setup infra for MCN container image
  (downstream) — To Do (Daniel Farrell)
  - Downstream image builds and registry config
  - Document downstream build process
- **CORENET-7084** — Setup downstream CI to deploy
  MCN — To Do (Daniel Farrell)
  - Set up downstream CI
  - Configure component deployment
- **CORENET-7085** — Enable upstream tests in downstream
  CI — To Do (Daniel Farrell)
  - No description yet, needs grooming

### Release, Integration, and Docs

- **CORENET-7089** — Release automation and
  versioning — To Do (Daniel Farrell)
  - No description yet, needs grooming
- **CORENET-7090** — CNO Integration — To Do (unassigned)
  - No description yet, needs grooming
- **CORENET-7156** — Add website to MCN — To Do (unassigned)
  - No description yet
- **CORENET-7157** — Add documentation — To Do (unassigned)
  - No description yet

## Related Issues (other parent epics)

### Under CORENET-7006 (Dev Preview epic)

- **CORENET-7039** — Create MCN repo in OVNK org — To Do
  (Vishal Thapar)
- **CORENET-7041** — Add Agentic AI workflows for MCN
  repo — To Do (Vishal Thapar)

### Under CORENET-6983 (Kube 1.36 Rebase, Blocker)

- **CORENET-7155** — Create agents to automate the
  bump — To Do (Daniel Farrell)

### ACM Project

- **ACM-25779** — Complete onboarding to new Konflux-build-catalog
  project [MCN] — New (Daniel Farrell)
  - Instructions at stolostron/konflux-build-catalog README

## My Assignments (Daniel Farrell) — Summary

11 open issues assigned to me across repo setup and automation:

1. CORENET-7078 — Container image infra (upstream)
2. CORENET-7079 — Container image infra (downstream)
3. CORENET-7081 — Makefiles and build system (upstream)
4. CORENET-7082 — Initial downstream repo setup
5. CORENET-7084 — Downstream CI
6. CORENET-7085 — Upstream tests in downstream CI
7. CORENET-7086 — Pre-merge testing automation
8. CORENET-7087 — Container image builds in CI
9. CORENET-7089 — Release automation and versioning
10. CORENET-7155 — Agents to automate kube bump
11. ACM-25779 — Konflux-build-catalog onboarding (ACM project)

Of these, 7085 and 7089 have no description and need grooming.
7090 (CNO Integration, unassigned) also needs grooming.

## Open Questions

- Upstream repo approval depends on CORENET-7077 (design proposal) —
  what's the timeline?
- CORENET-7085, 7089, 7090 have no description — need grooming
- How does CNO integration (7090) affect the repo setup sequence?
- ~~What existing Submariner automation patterns carry over vs need
  rethinking?~~ — Resolved. See tooling proposal
  (`mcn/2026-05-19-tooling-proposal.md`) and 50 project deep dives.

## Suggested Sequencing

1. CORENET-7077 (design proposal) must land first — gates upstream repo
2. CORENET-7080 (upstream repo scaffolding) once approved
3. CORENET-7081 (Makefiles/build system) and 7078 (container image
   infra) can follow in parallel
4. CORENET-7083 (Prow jobs) and 7086 (pre-merge testing) layer on
   top of build system
5. CORENET-7087 (CI container builds) depends on both Prow and image
   infra
6. CORENET-7082 (downstream repo) can start in parallel with upstream
7. CORENET-7079, 7084, 7085 (downstream CI/images/tests) follow
   downstream repo creation
8. CORENET-7089 (release automation) can be developed alongside once
   repos exist

## Jira Planning — Mapping Tooling Proposal to Stories

Based on the consolidated tooling proposal
(`mcn/2026-05-19-tooling-proposal.md`), here is what needs to happen
on Jira. No changes yet — this is the plan.

### Stories That Need Updated Descriptions

These stories exist but have vague or missing descriptions. They
should be updated with specific tooling from the proposal.

#### CORENET-7086 — Enable pre-merge testing automation

Current description mentions golangci-lint, gofmt, license headers,
API compat checks. Update to include the full Phase 1 linting stack:

- golangci-lint v2 with KAL plugin (15 linters, see proposal
  section 1)
- markdownlint-cli2, yamllint, shellcheck, hadolint, actionlint,
  zizmor (see proposal section 2)
- kubeconform + kube-linter for K8s manifest validation
- govulncheck, CodeQL, OSSF Scorecard, dependency-review-action
  (see proposal section 3)
- Ginkgo/Gomega + envtest for unit/integration tests
- `-shuffle=on`, `go mod tidy -diff` test flags
- Three AI review workflows (security, RBAC, release notes)
- conform (or PR title check) for commit message linting
- lychee for link checking

This is a large story. Consider creating subtasks (see below).

#### CORENET-7085 — Enable upstream tests in downstream CI

No description. Should cover:

- Running upstream unit tests in downstream Prow jobs
- Running upstream E2E tests on downstream builds
- Coverage reporting to Codecov
- go-test-coverage ratcheting

#### CORENET-7089 — Release automation and versioning

No description. Should cover:

- Conventional Commits enforcement (conform or PR title check)
- release-please (GitHub Action, not App) for versioning
- Per-PR changelog files (Contour pattern) with CI enforcement
- GoReleaser for binary builds (if MCN ships a CLI)
- GoReleaser dry-run on config changes
- Dependabot for GHA monthly + Go modules weekly
- Dependabot auto-fix (regenerate code on dep updates)
- Backport action (korthout/backport-action) for release branches
- Release artifact verification (diff release vs source)
- Fake release smoke test (daily `v9.9.9-fake` build)
- Cosign keyless signing for images
- Syft SBOM generation
- SLSA provenance attestations

#### CORENET-7090 — CNO Integration

No description. Not directly related to tooling but may need CI
jobs for integration testing with CNO.

### Stories That Need Subtasks

**CORENET-7086** is too large for a single story. Recommended
subtasks:

1. **Linting configs** — create all config files (.golangci.yml,
   .markdownlint.yml, .yamllint.yml, .shellcheckrc, .gitlint,
   .grype.yaml, .markdownlinkcheck.json, staticcheck.conf)
2. **GHA linting workflow** — create `.github/workflows/linting.yml`
   with 18+ parallel jobs
3. **GHA unit test workflow** — create `.github/workflows/unit.yml`
4. **GHA branch enforcement** — create `.github/workflows/branch.yml`
5. **GHA stale management** — create `.github/workflows/stale.yml`
6. **GHA periodic checks** — create `.github/workflows/periodic.yml`
7. **AI review workflows** — create 3 AI review workflow files
8. **KAL integration** — set up `.custom-gcl.yml` and KAL config
9. **Security scanning workflows** — govulncheck, CodeQL, Scorecard,
   zizmor, dependency-review, Gitleaks, harden-runner
10. **Dependabot config** — create `.github/dependabot.yml`

**CORENET-7081** subtasks:

1. **Makefile** — create standalone Makefile with standard targets
   (build, test, lint, images, manifests, generate, clean, help)
2. **`.github/env`** — centralized tool version pinning
3. **CLAUDE.md + AGENTS.md** — AI agent instructions
4. **Lefthook config** — pre-commit hooks (lefthook.yml)
5. **SECURITY-INSIGHTS.yml** — OpenSSF security metadata

**CORENET-7078** subtasks:

1. **ko configuration** — `.ko.yaml` for upstream image builds
2. **Upstream Dockerfile** — multi-stage for downstream/Konflux
3. **Image push workflow** — `.github/workflows/release.yml`
4. **Multi-arch build config** — amd64 + arm64 at minimum

**CORENET-7089** subtasks:

1. **Conventional Commits setup** — conform config or PR title check
2. **release-please workflow** — automated versioning
3. **Per-PR changelog system** — CI enforcement + release assembly
4. **Supply chain security** — Cosign + Syft + SLSA in release
5. **Backport automation** — label-driven cherry-pick

### New Stories to Propose

These are capabilities from the proposal that don't map cleanly
to any existing Jira story:

1. **Governance files** — LICENSE, README, CONTRIBUTING, CODEOWNERS,
   .gitignore, .gitattributes, PR template, issue templates. Could
   be a subtask of CORENET-7080 or a new story.

2. **CRD validation CI** — crdify for breaking changes, go-apidiff
   for Go API compat, CEL validation matrix, CRD drift verification.
   Could be subtask of CORENET-7086 or a new story.

3. **E2E test framework** — KIND cluster setup, Ginkgo E2E suite,
   upgrade E2E, system validation script. Probably needs its own
   story (separate from CORENET-7086 which is pre-merge testing).

4. **Developer experience** — devcontainer.json, Lefthook setup,
   echo.% Makefile introspection. Could be subtask of CORENET-7081.

5. **CI workflow patterns** — dorny/paths-filter, draft-aware matrix,
   workflow failure issue tracker, workflow telemetry, composite
   result pattern, write-on-merge cache. These are improvements to
   the CI workflows, not separate stories. Add as acceptance criteria
   to CORENET-7086.

### Stories That Are Ready As-Is

These stories have adequate descriptions that align with the
tooling proposal:

- **CORENET-7078** — container image infra (upstream). Aligns with
  proposal section 6 (ko, multi-arch). May need subtasks but
  description is sufficient.
- **CORENET-7081** — Makefiles and build system. Aligns with
  proposal. May need subtasks.
- **CORENET-7082** — downstream repo setup. Process-focused, not
  tooling-heavy.
- **CORENET-7083** — Prow jobs. Assigned to Prachi. Aligns with
  proposal section 8 (GHA + Prow split).
- **CORENET-7084** — downstream CI. Process-focused.

### Grooming Priority

1. **CORENET-7086** — highest priority. This is the "CI that
   validates all contributions" story. Needs the most detailed
   description and subtasks from the tooling proposal.
2. **CORENET-7089** — second priority. Release automation needs
   detailed acceptance criteria from the proposal.
3. **CORENET-7085** — third. Needs any description at all.
4. **CORENET-7090** — lower. CNO integration is blocked on other
   work anyway.

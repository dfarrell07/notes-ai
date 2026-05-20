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
- How does CNO integration (7090) affect the repo setup sequence?
- Stories needing grooming: see Jira Planning section below

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

## Jira Planning — Complete Structure

Comprehensive view of all stories, their scope, proposed
subtasks, and gaps. Based on the tooling proposal and review
of actual Jira descriptions.

### Existing Stories — Scope Clarification

#### Upstream

| Jira | Scope | Status | Subtasks Needed? |
| --- | --- | --- | --- |
| 7077 | Design proposal (gates everything) | To Do (Yossi) | No |
| 7080 | Repo scaffolding (dirs, go.mod, LICENSE) | To Do (unassigned) | No |
| 7081 | Makefile + build system | To Do (Daniel) | Yes (5) |
| 7078 | Container image infra (upstream) | To Do (Daniel) | Yes (4) |
| 7083 | Prow job configuration | To Do (Prachi) | No |
| 7086 | Pre-merge testing / GHA CI | To Do (Daniel) | Yes (10) |
| 7087 | CI container image builds + push | To Do (Daniel) | No |
| 7089 | Upstream release process | To Do (Daniel) | Yes (4) |

#### Downstream (under CORENET-7067)

| Jira | Scope | Status | Subtasks Needed? |
| --- | --- | --- | --- |
| 7082 | Create openshift/mcn, sync from upstream | To Do (Daniel) | No |
| 7079 | Konflux build infra (Dockerfiles, Tekton) | To Do (Daniel) | Yes (3) |
| 7084 | Downstream CI (Prow on downstream builds) | To Do (Daniel) | No |
| 7085 | Run upstream tests on downstream builds | To Do (Daniel) | No |
| ACM-25779 | Konflux-build-catalog onboarding | New (Daniel) | No |

#### Other

| Jira | Scope | Status | Notes |
| --- | --- | --- | --- |
| 7090 | CNO integration | To Do (unassigned) | Needs grooming |
| 7156 | Website | To Do (unassigned) | Later |
| 7157 | Documentation | To Do (unassigned) | Later |
| 7155 | Kube bump automation agents | To Do (Daniel) | Different epic |
| 7041 | Agentic AI workflows | To Do (Vishal) | Different epic |

### Proposed Subtasks

#### CORENET-7080 subtasks (repo scaffolding)

1. Governance files (LICENSE, README, CONTRIBUTING, CODEOWNERS)
2. PR template + issue templates
3. .gitignore, .gitattributes

#### CORENET-7081 subtasks (build system)

1. Makefile with standard targets
2. `.github/env` for tool version pinning
3. CLAUDE.md + AGENTS.md
4. Lefthook pre-commit config
5. SECURITY-INSIGHTS.yml

#### CORENET-7078 subtasks (upstream images)

1. ko configuration (`.ko.yaml`)
2. Upstream Dockerfile (multi-stage, for downstream reference)
3. GHA image push workflow (push on merge to main)
4. Multi-arch config (amd64 + arm64)

#### CORENET-7086 subtasks (pre-merge CI)

1. Linting config files (.golangci.yml, .markdownlint.yml,
   .yamllint.yml, .shellcheckrc, .grype.yaml, staticcheck.conf)
2. GHA linting workflow (18+ parallel jobs)
3. GHA unit test workflow
4. GHA branch enforcement + stale management
5. GHA periodic checks (weekly link check)
6. AI review workflows (security, RBAC, release notes)
7. KAL integration (.custom-gcl.yml + .golangci-kal.yml)
8. Security scanning (govulncheck, CodeQL, Scorecard, zizmor,
   dependency-review, Gitleaks, harden-runner)
9. Dependabot config
10. CRD validation CI (codegen diff, crdify, go-apidiff)

#### CORENET-7089 subtasks (upstream release)

1. Conventional Commits enforcement (conform or PR title check)
2. release-please workflow (GitHub Action)
3. Per-PR changelog system (Contour pattern, CI enforced)
4. Backport automation (korthout/backport-action)

#### CORENET-7079 subtasks (downstream images)

1. Konflux Dockerfiles (UBI9, FIPS, multi-arch)
2. Tekton pipeline configs (.tekton/)
3. RPM lockfiles (if needed)

### New Stories to Create

#### NEW: Downstream/Konflux release pipeline

Parent: CORENET-7067. Covers the full Konflux release process
that has no existing story:

- Konflux tenant configuration (konflux-release-data overlays)
- ReleasePlanAdmission setup
- Enterprise Contract policy
- Stage release workflow (Release CRs, snapshots)
- Prod release workflow
- FBC catalog setup and management
- OLM bundle generation and validation
- Image signing (Cosign keyless) at release time
- SBOM generation (Syft) and attestation
- SLSA provenance per image

This is Phase 4 work. Depends on 7079 (build infra exists) and
ACM-25779 (catalog onboarding). Submariner's equivalent is a
20-step process — this will need its own subtasks.

#### NEW: E2E test framework

Parent: CORENET-7067. Separate from 7086 (pre-merge CI):

- KIND cluster setup for local E2E
- Ginkgo E2E test suite structure
- Upgrade E2E (N-1 to N)
- System validation script (deployment correctness check)

This is Phase 3 work. Depends on 7081 (build system) and 7086
(unit tests exist).

### Stories That Don't Need Changes

- **7077** — design proposal, gates everything, clear
- **7082** — downstream repo creation, clear
- **7083** — Prow jobs, assigned to Prachi, clear
- **7084** — downstream CI, clear (thin description but scope
  is obvious)
- **7087** — upstream CI image builds, clear

### Actions Summary

**Update descriptions** (add comment with proposed scope):

1. CORENET-7086 — add full Phase 1 CI tooling list
2. CORENET-7089 — scope to upstream release, add tooling
3. CORENET-7085 — add initial scope (test reuse, coverage)

**Create new stories**:

1. Downstream/Konflux release pipeline (Phase 4)
2. E2E test framework (Phase 3)

**Create subtasks** (after descriptions are agreed):

1. 10 subtasks under 7086
2. 5 subtasks under 7081
3. 4 subtasks under 7078 and 7089
4. 3 subtasks under 7079

**Grooming priority**: 7086 first, then 7089, then 7085, then
new stories.

Draft comments for 7086, 7089, 7085 are in
`mcn/2026-05-19-jira-draft-comments.md`.

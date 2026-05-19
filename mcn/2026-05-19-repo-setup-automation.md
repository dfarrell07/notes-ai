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

10 open issues assigned to me across repo setup and automation:

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
11. ACM-25779 — Konflux-build-catalog onboarding

Of these, 7085, 7089, and 7090 have no description and need grooming.

## Open Questions

- Upstream repo approval depends on CORENET-7077 (design proposal) —
  what's the timeline?
- CORENET-7085, 7089, 7090 have no description — need grooming
- How does CNO integration (7090) affect the repo setup sequence?
- What existing Submariner automation patterns carry over vs need
  rethinking?

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

---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [repo-setup, automation, ci, openshift, multicluster-networking, jira]
---

# New Repo Setup and Automation for OpenShift Multicluster Networking

Planning notes for bootstrapping the OpenShift Multicluster Networking (MCN)
project repositories. Goal: get automation and infrastructure in place before
any application code lands.

## Jira Issues — Repo Setup and Automation

Parent epic: **CORENET-7067** — "Create new repos for MCN in upstream and
downstream" (In Progress, Major priority)

### Upstream (under CORENET-7067)

- **CORENET-7078** — Setup infra for MCN container image registry and
  publishing (upstream) — To Do
  - Design container image structure
  - Build config, registry location, tagging strategy
  - Configure multiarch builds
- **CORENET-7081** — Add Makefiles and build system to upstream MCN
  repository — To Do
  - Makefile with standard targets
  - Container builds
  - Build tags and versioning strategy
  - Code generation tools (controller-gen, client-gen)
- **CORENET-7086** — Enable pre-merge testing automation — To Do
  - Unit test execution in pre-merge
  - Code coverage reporting
  - Linting and formatting (golangci-lint, gofmt)
  - License header verification
  - API compatibility checks

### Downstream (under CORENET-7067)

- **CORENET-7079** — Setup infra for MCN container image
  (downstream) — To Do
  - Downstream image builds and registry config
  - Document downstream build process
- **CORENET-7082** — Initial repository setup downstream for MCN — To Do
  - Create openshift/mcn repository
  - Configure downstream build system
  - Set up sync automation from upstream
  - Document downstream contribution workflow
- **CORENET-7084** — Setup downstream CI to deploy MCN — To Do
  - Set up downstream CI
  - Configure component deployment

### Release and Versioning (under CORENET-7067)

- **CORENET-7089** — Release automation and versioning — To Do
  - No description yet, needs grooming

### Related (not under CORENET-7067)

- **CORENET-7155** — Create agents to automate the bump — To Do
  - Parent: CORENET-6983 (Rebase Kube to 1.36, Blocker priority)
- **ACM-25779** — Complete onboarding to new Konflux-build-catalog
  project [MCN] — New
  - Instructions at stolostron/konflux-build-catalog README

## Key Areas to Address

### CI/CD Pipeline

- Decide on CI system (Prow, Konflux, GitHub Actions, or combination)
- Set up linting (Go, YAML, Markdown, shell)
- Container image builds and registry push
- Unit and integration test harnesses

### Repository Scaffolding

- OWNERS / CODEOWNERS files
- CLAUDE.md for Claude Code conventions
- Makefile or Taskfile with standard targets (build, test, lint, verify)
- Go module initialization and dependency management
- Dockerfile(s) with version labels

### Code Quality Gates

- Pre-commit hooks or CI checks for:
  - Go formatting (gofmt, goimports)
  - Go linting (golangci-lint)
  - govulncheck for vulnerability scanning
  - License header verification
  - Markdown linting
- Branch protection rules on main

### Release Automation

- Versioning strategy (semver, release branches)
- Changelog generation
- Release note tooling
- Container image tagging strategy

### Documentation

- README with project overview, quickstart, contributing guide
- Developer setup instructions
- Architecture decision records (ADRs) if applicable

## Open Questions

- Which repos need to be created? (operator, agent, API, e2e tests, docs?)
- Upstream vs downstream repo split?
- Konflux onboarding timeline and requirements?
- What existing Submariner automation patterns carry over vs need
  rethinking?
- CORENET-7089 has no description — what's the release/versioning plan?

## Next Steps

- Enumerate the specific repos and their purposes
- Draft the Makefile target list
- Decide on CI system and write initial pipeline configs
- Groom CORENET-7089 (release automation) — add acceptance criteria
- Start with CORENET-7082 (downstream repo creation) as the foundation

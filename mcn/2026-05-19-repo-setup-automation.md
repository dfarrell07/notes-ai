---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [repo-setup, automation, ci, openshift, multicluster-networking]
---

# New Repo Setup and Automation for OpenShift Multicluster Networking

Planning notes for bootstrapping the OpenShift Multicluster Networking (MCN)
project repositories. Goal: get automation and infrastructure in place before
any application code lands.

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
- What existing Submariner automation patterns carry over vs need rethinking?

## Next Steps

- Enumerate the specific repos and their purposes
- Draft the Makefile target list
- Decide on CI system and write initial pipeline configs

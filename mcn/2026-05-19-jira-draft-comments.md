---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [jira, draft-comments, ci, release, testing]
---

# Draft Jira Comments

## CORENET-7086 — Enable Pre-merge testing automation

---

Researched CI patterns across ~50 K8s projects. Planning to set
up GHA workflows covering:

- golangci-lint v2 (strict config, plus KAL for CRD conventions)
- Standard non-Go linting (markdownlint, yamllint, shellcheck,
  hadolint, kubeconform, kube-linter)
- actionlint + zizmor for GHA workflow validation
- Security scanning (govulncheck, CodeQL, OSSF Scorecard,
  Gitleaks, dependency-review-action)
- Ginkgo/Gomega + envtest unit tests
- AI-powered non-blocking PR review (security, RBAC, release
  notes)

GHA for linting/unit/images, Prow for cloud E2E (CORENET-7083).

Will split into subtasks — this is too much for one PR.

---

## CORENET-7089 — Release Automation and Versioning

---

Proposing for upstream release process:

- Conventional Commits + release-please for automated versioning
  and changelogs
- Per-PR changelog files (Contour pattern) assembled at release
  time
- Backport automation via labels

Open to other approaches.

---

## CORENET-7085 — Enable upstream tests in Downstream CI

---

Proposing:

- Run upstream unit + E2E tests as Prow jobs on openshift/mcn
- Coverage tracking with per-package ratcheting
- Downstream-specific verification (UBI9/FIPS build works, CRD
  compat)

Blocked on 7082 (downstream repo), 7086 (upstream tests), 7083
(Prow config).

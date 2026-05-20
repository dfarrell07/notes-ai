---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [jira, draft-comments, ci]
---

# Draft Jira Comment

Only CORENET-7086 needs a comment right now. The others (7089,
7085) just need description updates when the time comes — they're
blocked on earlier work anyway.

## CORENET-7086 — Enable Pre-merge testing automation

---

Planning to set up GHA workflows covering:

- golangci-lint v2 (strict config, plus KAL for CRD conventions)
- Standard non-Go linting (markdownlint, yamllint, shellcheck,
  hadolint, kubeconform, kube-linter)
- actionlint + zizmor for GHA workflow validation
- Security scanning (govulncheck, CodeQL, OSSF Scorecard,
  Gitleaks, dependency-review-action)
- Ginkgo/Gomega + envtest unit tests
- AI-powered non-blocking PR review (security, RBAC, release
  notes)
- Dependabot for GHA and Go module updates

GHA for linting/unit/images, Prow for cloud E2E (CORENET-7083).

Will split into subtasks — this is too much for one PR.

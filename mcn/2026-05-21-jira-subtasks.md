---
date: 2026-05-21T00:00:00Z
topic: mcn
tags: [jira, subtasks, ci, tooling]
---

# CORENET-7086 Subtasks — Draft Titles and Descriptions

28 subtasks, one per tool. All blocked on CORENET-7080 (repo
creation). Parent: CORENET-7086 (Enable pre-merge testing
automation).

## 1. Add golangci-lint v2

Set up golangci-lint v2 with a strict config enabling 60+
linters. Use `default: none` with explicit enable list. Include
formatters (gci, gofmt, gofumpt, goimports). Add `.golangci.yml`
and a GHA linting workflow job.

## 2. Add kube-api-linter (KAL)

Set up KAL as a custom golangci-lint plugin via `.custom-gcl.yml`.
Enforces K8s API conventions on CRD types (json tags, optional/
required markers, no bools, no phase, status subresource). Scope
to `api/` directory. Add `.golangci-kal.yml` and a GHA workflow
job.

## 3. Add markdownlint-cli2

Add `.markdownlint.yml` config and a GHA workflow job to lint
markdown files.

## 4. Add yamllint

Add `.yamllint.yml` config and a GHA workflow job to lint YAML
files. Ignore generated directories and GHA workflow truthy
values.

## 5. Add shellcheck

Add `.shellcheckrc` config and a GHA workflow job to lint shell
scripts.

## 6. Add shfmt

Add a GHA workflow job to verify shell script formatting.

## 7. Add hadolint

Add a GHA workflow job to lint Dockerfiles for best practices.

## 8. Add actionlint

Add a GHA workflow job to validate GHA workflow files for syntax
and type errors. Complements zizmor (correctness vs security).

## 9. Add zizmor

Add a GHA workflow job to audit GHA workflow files for security
issues (template injection, unpinned actions, secrets exposure).
Output SARIF to GitHub Security tab.

## 10. Add kubeconform

Add a GHA workflow job to validate K8s manifests against schemas.
Supports CRDs.

## 11. Add kube-linter

Add a GHA workflow job to check K8s manifests for security and
best-practice violations (running as root, missing resource
limits, missing probes, etc.).

## 12. Add lychee

Add lychee for markdown link checking. PR workflow checks
modified files only. Weekly periodic workflow checks all files
and auto-creates a GitHub issue if broken links found.

## 13. Add govulncheck

Add a GHA workflow job running govulncheck for Go dependency
vulnerability scanning with SARIF output. Uses symbol-level
reachability analysis to minimize false positives.

## 14. Add CodeQL

Add a GHA workflow running CodeQL for static application security
testing (SAST) on Go code. Upload SARIF to GitHub Code Scanning.

## 15. Add OSSF Scorecard

Add a weekly workflow running OSSF Scorecard for supply chain
security assessment. Upload SARIF to GitHub Security tab.

## 16. Add dependency-review-action

Add GitHub's dependency-review-action to PR checks to block PRs
that introduce known-vulnerable dependencies.

## 17. Add Gitleaks

Add a GHA workflow job running Gitleaks for secrets scanning in
PR diffs. Output SARIF to GitHub Security tab.

## 18. Add harden-runner

Add step-security/harden-runner as the first step in all GHA
workflows for network egress monitoring. Start with audit mode.

## 19. Add Dependabot

Add `.github/dependabot.yml` for GitHub Actions (monthly) and
Go modules (weekly) dependency updates. Group k8s.io dependencies
together.

## 20. Add Ginkgo/envtest unit test workflow

Add a GHA workflow running unit tests with Ginkgo/Gomega and
envtest for controller integration tests. Use `-shuffle=on`
to catch ordering-dependent tests.

## 21. Add Codecov

Add Codecov integration for PR-level coverage reporting with
line-level comments and coverage diff.

## 22. Add go-test-coverage

Add go-test-coverage for per-package coverage thresholds that
ratchet up over time.

## 23. Add lichen dependency license compliance

Add a GHA workflow job running lichen against compiled binaries
to verify all dependencies have approved licenses (Apache-2.0,
MIT, BSD, ISC, etc.). Same allowlist as Submariner.

## 24. Add AI security review workflow

Add a GHA workflow using anthropics/claude-code-action for
non-blocking security review on PRs. Checks for privilege
escalation, secrets handling, security context changes,
hardcoded credentials. Read-only tools, sticky comments.

## 25. Add AI RBAC review workflow

Add a GHA workflow using anthropics/claude-code-action for
detailed RBAC change analysis. Path-filtered to only trigger
on RBAC-related file changes. Verifies least-privilege against
controller source code.

## 26. Add AI release notes workflow

Add a GHA workflow using anthropics/claude-code-action to suggest
release note text when a PR has user-facing changes. Posts
suggested text as a PR comment.

## 27. Add branch enforcement workflow

Add a GHA workflow enforcing that all PRs target main or
release-* branches.

## 28. Add stale issue/PR workflow

Add a GHA workflow closing stale issues after 120 days and PRs
after 14 days. Exempt labels: confirmed, security.

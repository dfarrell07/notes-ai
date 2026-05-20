---
date: 2026-05-21T00:00:00Z
topic: mcn
tags: [jira, subtasks, ci, tooling]
---

# CORENET-7086 Subtasks — Draft Titles and Descriptions

29 subtasks, one per tool. All blocked on CORENET-7080 (repo
creation). Parent: CORENET-7086 (Enable pre-merge testing
automation).

## 1. Add golangci-lint v2

Set up golangci-lint v2 with a strict config based on
Submariner's. Use `default: none` with explicit enable list.
Enable importas, modernize, funcorder, recvcheck, iface
(identical only), depguard, forbidigo, promlinter, exhaustive,
goheader, gosec. Add `.golangci.yml` and a linting GHA workflow
job.

## 2. Add kube-api-linter (KAL)

Set up KAL as a custom golangci-lint plugin via `.custom-gcl.yml`.
Enable selectively: jsontags, optionalorrequired, requiredfields,
defaults, statussubresource, nobools. Scope to `api/` directory.
Add `.golangci-kal.yml` and a separate linting workflow job.

## 3. Add markdownlint-cli2

Add `.markdownlint.yml` config (140 char line length). Add a
linting workflow job running `npx markdownlint-cli2`.

## 4. Add yamllint

Add `.yamllint.yml` config (140 char lines, truthy ignore for
GHA workflows, ignore generated dirs). Add a linting workflow
job via `pip install yamllint`.

## 5. Add shellcheck

Add `.shellcheckrc` config (disable SC1090, SC2154). Add a
linting workflow job running shellcheck on all `.sh` files.

## 6. Add shfmt

Add a linting workflow job running `shfmt -d` to verify shell
script formatting.

## 7. Add hadolint

Add a linting workflow job running hadolint on all Dockerfiles.

## 8. Add actionlint

Add a linting workflow job running actionlint on all GHA workflow
files. Checks syntax, expression types, action inputs/outputs.
Complements zizmor (correctness vs security).

## 9. Add zizmor

Add a linting workflow job running zizmor on all GHA workflow
files. 24 security audit rules covering template injection,
unpinned actions, secrets exposure. Output SARIF to GitHub
Security tab.

## 10. Add kubeconform

Add a linting workflow job running kubeconform for K8s manifest
schema validation. Replaces the abandoned kubeval. Supports CRDs.

## 11. Add kube-linter

Add a linting workflow job running kube-linter for K8s manifest
security checks (40+ rules: running as root, missing resource
limits, missing probes, latest tag, etc.).

## 12. Add lychee

Add a periodic workflow running lychee for link checking in
markdown files. Weekly cron on main, modified-files-only on PRs.
Replaces the deprecated markdown-link-check.

## 13. Add govulncheck

Add a linting workflow job running govulncheck for Go
vulnerability scanning with SARIF output. Official Go team tool
with symbol-level reachability analysis.

## 14. Add CodeQL

Add a security workflow running CodeQL variant analysis for Go.
Upload SARIF results to GitHub Code Scanning.

## 15. Add OSSF Scorecard

Add a weekly + push workflow running OSSF Scorecard for supply
chain security assessment. Upload SARIF to GitHub Security tab.
Publishes a public scorecard badge.

## 16. Add dependency-review-action

Add a PR workflow step running GitHub's dependency-review-action
to block PRs that introduce known-vulnerable dependencies.

## 17. Add Gitleaks

Add a security workflow job running Gitleaks for secrets scanning
in PR diffs. MIT license, replaces TruffleHog (AGPL).

## 18. Add harden-runner

Add step-security/harden-runner to GHA workflows for network
egress monitoring. Start with `egress-policy: audit` mode.

## 19. Add Dependabot

Add `.github/dependabot.yml` config for GitHub Actions (monthly)
and Go modules (weekly) dependency updates. Group k8s.io
dependencies together.

## 20. Add Ginkgo/envtest unit test workflow

Add a GHA workflow running Ginkgo/Gomega unit tests with envtest
for controller integration tests. Use `-shuffle=on` and
`go mod tidy -diff` flags.

## 21. Add Codecov

Add Codecov integration for PR-level coverage reporting with
line-level comments.

## 22. Add go-test-coverage

Add go-test-coverage (vladopajic/go-test-coverage) for per-package
coverage ratcheting. Coverage can only go up, never down.

## 23. Add lichen dependency license compliance

Add a linting workflow job running lichen (or go-licenses) against
compiled binaries to verify all dependencies have CNCF-approved
licenses. Use the same allowlist as Submariner (Apache-2.0, MIT,
BSD-2, BSD-3, ISC, etc.).

## 24. Add AI security review workflow

Add a GHA workflow using anthropics/claude-code-action for
non-blocking security review on PRs. Check for RBAC changes,
privilege escalation, secrets handling, security context changes,
hardcoded credentials. Confidence scoring, read-only tools,
sticky comments.

## 25. Add AI RBAC review workflow

Add a GHA workflow using anthropics/claude-code-action for
detailed RBAC change analysis. Path-filtered to only trigger on
RBAC-related file changes (config/rbac/**, *_types.go,
controller*.go). Least-privilege verification against controller
source.

## 26. Add AI release notes workflow

Add a GHA workflow using anthropics/claude-code-action to suggest
release note text when a PR warrants one. Detects API/CRD changes,
breaking changes, new features, deprecations. Posts suggested text
as a PR comment.

## 27. Add branch enforcement workflow

Add a GHA workflow enforcing that all PRs target main or
release-* branches.

## 28. Add stale issue/PR workflow

Add a GHA workflow closing stale issues after 120 days and PRs
after 14 days. Exempt labels: confirmed, security.

## 29. Add periodic link check workflow

Add a weekly GHA workflow running lychee across all markdown files
(full scan, not just modified). Auto-create GitHub issue if broken
links found.

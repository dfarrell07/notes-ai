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
linters. Use v2 config format with `default: none` and explicit
enable list. Configure formatters separately (gci, gofumpt,
goimports). Add `.golangci.yml` and a GHA linting workflow job.

## 2. Add kube-api-linter (KAL)

Set up KAL as a custom golangci-lint plugin via `.custom-gcl.yml`.
Enforces K8s API conventions on CRD types (json tags, optional/
required markers, no bools, no phase, status subresource). Scope
to `api/` directory. Requires building a custom golangci-lint
binary in CI before linting.

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
Configure formatting style (indent width, etc.) via
`.editorconfig` or flags.

## 7. Add hadolint

Add a GHA workflow job to lint Dockerfiles for best practices.
Consider a `.hadolint.yaml` config for rule suppression and
trusted registries.

## 8. Add actionlint

Add a GHA workflow job to validate GHA workflow files for syntax
and type errors. Also checks shell in `run:` steps via embedded
shellcheck. Complements zizmor (correctness vs security).

## 9. Add zizmor

Add a GHA workflow job to audit GHA workflow files for security
issues (template injection, unpinned actions, secrets exposure).
Output SARIF to GitHub Security tab.

## 10. Add kubeconform

Add a GHA workflow job to validate K8s manifests against schemas.
Configure schema sources for any custom CRDs. May need a build
step to render manifests first if using kustomize.

## 11. Add kube-linter

Add a GHA workflow job to check K8s manifests for security and
best-practice violations (running as root, missing resource
limits, missing probes, etc.). Consider a `.kube-linter.yml`
config to tune rules for operator manifests.

## 12. Add lychee

Add lychee for markdown link checking. PR workflow checks
modified files only. Weekly periodic workflow checks all files
and auto-creates a GitHub issue if broken links found.

## 13. Add govulncheck

Add a GHA workflow job running govulncheck for Go dependency
vulnerability scanning with SARIF output. Uses symbol-level
reachability analysis to minimize false positives. Ensure build
tags match actual build configuration.

## 14. Add CodeQL

Add a GHA workflow running CodeQL for static application security
testing (SAST) on Go code. Upload SARIF to GitHub Code Scanning.
Main value is taint-tracking and injection-path analysis beyond
what golangci-lint catches.

## 15. Add OSSF Scorecard

Add a weekly workflow running OSSF Scorecard for supply chain
security assessment. Upload SARIF to GitHub Security tab. Initial
scores will improve as other CI tools are added.

## 16. Add dependency-review-action

Add GitHub's dependency-review-action to PR checks to block PRs
that introduce known-vulnerable dependencies. Only checks the PR
diff, not existing dependencies.

## 17. Add Gitleaks

Add a GHA workflow job running Gitleaks for secrets scanning in
PR diffs. Output SARIF to GitHub Security tab. Add a
`.gitleaks.toml` config if allowlisting is needed for test
fixtures.

## 18. Add harden-runner

Add step-security/harden-runner as the first step in all GHA
workflow jobs for network egress monitoring. Start with audit
mode. Note: sends telemetry to StepSecurity SaaS backend.

## 19. Add Dependabot

Add `.github/dependabot.yml` for GitHub Actions (monthly) and
Go modules (weekly) dependency updates. Group k8s.io and
sigs.k8s.io dependencies together.

## 20. Add Ginkgo/envtest unit test workflow

Add a GHA workflow running unit tests with Ginkgo/Gomega and
envtest for controller integration tests. Includes setup-envtest
for downloading control plane binaries. Use `-shuffle=on` and
`-race` flags.

## 21. Add Codecov

Add Codecov integration for PR-level coverage reporting with
line-level comments and coverage diff. Requires CODECOV_TOKEN
secret. Provides visibility; go-test-coverage provides
enforcement.

## 22. Add go-test-coverage

Add go-test-coverage for per-package coverage thresholds that
ratchet up over time. Provides enforcement; Codecov provides
visibility. Depends on unit test workflow producing coverage
output.

## 23. Add lichen dependency license compliance

Add a GHA workflow job running lichen against compiled binaries
to verify all dependencies have approved licenses (Apache-2.0,
MIT, BSD, ISC, etc.). Same allowlist as Submariner. Requires a
build step before scanning.

## 24. Add AI security review workflow

Add a GHA workflow using anthropics/claude-code-action for
non-blocking security review on PRs. Checks for privilege
escalation, secrets handling, security context changes,
hardcoded credentials. Read-only tools, updates existing comment
on each push. Requires ANTHROPIC_API_KEY secret. Skip gracefully
on forks.

## 25. Add AI RBAC review workflow

Add a GHA workflow using anthropics/claude-code-action for
detailed RBAC change analysis. Path-filtered to trigger on
RBAC-related file changes (config/rbac/, kubebuilder RBAC
markers in Go source). Verifies least-privilege against
controller source code. Skip gracefully on forks.

## 26. Add AI release notes workflow

Add a GHA workflow using anthropics/claude-code-action to suggest
release note text when a PR has user-facing changes (API/CRD
changes, new features, breaking changes). Posts suggested text
as a PR comment. Skip gracefully on forks.

## 27. Add branch enforcement workflow

Add a GHA workflow enforcing that all PRs target main or
release-* branches.

## 28. Add stale issue/PR workflow

Add a GHA workflow labeling stale issues after 120 days and PRs
after 14 days. Exempt labels: confirmed, security. Consider
starting with labeling only (no auto-close) for a new project.

---
date: 2026-05-21T00:00:00Z
topic: mcn
tags: [jira, subtasks, ci, tooling]
---

# CORENET-7086 Subtasks — Draft Titles and Descriptions

26 subtasks, one per tool (#18 and #27 dropped). Parent:
CORENET-7086 (Enable pre-merge testing automation). Assignee:
dfarrell@redhat.com. Component: MCN (if subtask screen allows).

### Dropped tools

- **harden-runner (#18)**: Third-party SaaS with kernel-level
  access to every CI job. Three bypass CVEs in 14 months
  (DNS-over-TCP gave zero detections). Poor disclosure
  handling. GitHub's native egress firewall (GA ~Q4 2026) is
  the better path. Use SHA-pinned actions and explicit
  permissions instead.
- **Branch enforcement (#27)**: Use GitHub rulesets, not a GHA
  workflow. Premature until release branches exist.

### AI review: post-merge, not PR-triggered

Subtasks 23-25 (security, RBAC, release notes) run post-merge
(on push to main), not on PRs. Reasons:

- **Fork PRs can't access secrets.** Standard open-source
  contribution model is fork-based PRs. `on: pull_request`
  blocks secrets from forks, so AI review simply won't run on
  external contributions. `on: pull_request_target` exposes
  secrets to fork code — dangerous.
- **API abuse.** PR-triggered review lets anyone open PRs to
  burn API credits. No built-in rate limiting in
  claude-code-action.
- **Prompt injection.** Anthropic's own docs say the action
  "is not hardened against prompt injection attacks and should
  only be used to review trusted PRs." Merged code is trusted.
- **Post-merge is sufficient.** Security and RBAC findings
  after merge can be fixed in follow-up PRs. Release notes
  are inherently post-merge.
- **Local development.** CLAUDE.md guidance lets developers
  run the same checks locally via Claude Code before pushing,
  catching issues before they reach main.

## 1. Add golangci-lint v2 — CREATED (CORENET-7173)

Set up golangci-lint v2 with a strict curated config. Use v2
config format (`version: "2"`) with `default: none` and explicit
enable list. Configure formatters separately (gci, gofumpt,
goimports). Add `.golangci.yml` and a GHA linting workflow job.

## 2. Add kube-api-linter (KAL)

Set up KAL as a golangci-lint module via `.custom-gcl.yml`.
Enforces K8s API conventions on CRD types (json tags, optional/
required markers, no bools, no phase, status subresource). Scope
to `api/` directory. Requires `golangci-lint custom` to build a binary that
includes KAL before linting.

## 3. Add markdownlint-cli2

Add `.markdownlint.yml` config (line-length 140 to match Go)
and a GHA workflow job to lint markdown files.

## 4. Add yamllint

Add `.yamllint.yml` config (line-length 140, truthy ignore
scoped to `.github/workflows/`) and a GHA workflow job with
`--strict`. Ignore generated directories.

## 5. Add shellcheck

Add `.shellcheckrc` config and a GHA workflow job to lint
shell scripts. Use shebang-based auto-discovery and `-x` to
follow sourced files.

## 6. Add shfmt

Add a GHA workflow job to verify shell script formatting
using `shfmt -d` (diff mode). Configure formatting style
(indent width, case indent) via `.editorconfig`.

## 7. Add hadolint

Add a GHA workflow job to lint Dockerfiles for best practices.
Add a `.hadolint.yaml` config for rule suppression and
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

Add a GHA workflow job to validate K8s manifests (RBAC,
Deployments, sample CRs) against schemas. Configure schema
sources for custom CRDs. Include a kustomize render step
before validation if needed.

## 11. Add kube-linter

Add a GHA workflow job to check K8s manifests for security and
best-practice violations (running as root, missing resource
limits, missing probes, etc.). Add a `.kube-linter.yaml`
config to tune rules for operator manifests.

## 12. Add lychee

Add lychee for markdown link checking. PR workflow checks
modified files only. Weekly periodic workflow checks all
files and opens a GitHub issue on failure via
`peter-evans/create-issue-from-file`.

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
scores will improve as branch protection, review policies,
and security tooling are configured.

## 16. Add dependency-review-action

Add GitHub's dependency-review-action to PR checks to block PRs
that introduce known-vulnerable dependencies. Only checks the PR
diff, not existing dependencies.

## 17. Add Gitleaks

Add a GHA workflow job running Gitleaks for secrets scanning in
PR diffs. Output SARIF to GitHub Security tab. Add a
`.gitleaks.toml` config if allowlisting is needed for test
fixtures. Evaluate Betterleaks (MIT, drop-in replacement by
same author) at implementation time.

## 18. Add Dependabot

Add `.github/dependabot.yml` for GitHub Actions (monthly) and
Go modules (weekly) dependency updates. Group k8s.io and
sigs.k8s.io dependencies together.

## 19. Add Ginkgo/envtest test workflow

Add a GHA workflow running unit and integration tests with
Ginkgo/Gomega and envtest. Includes setup-envtest for
downloading control plane binaries. Use `--randomize-all`
and `-race` flags.

## 20. Add Codecov

Add Codecov integration for PR-level coverage reporting with
line-level comments and coverage diff. CODECOV_TOKEN may be
required depending on org settings (tokenless available for
public repos with v5+ action). Provides visibility;
go-test-coverage provides enforcement.

## 21. Add go-test-coverage

Add go-test-coverage for per-package coverage thresholds.
Use its diff feature to prevent coverage regressions vs. the
base branch. Provides enforcement; Codecov provides
visibility. Depends on test workflow producing coverage
output.

## 22. Add go-licenses dependency license compliance

Add a GHA workflow job running google/go-licenses to verify
all dependencies have approved licenses (Apache-2.0, MIT,
BSD, ISC, etc.). Same allowlist approach as Submariner. Scans
source modules directly — no build step required.

## 23. Add AI security review automation

Add a post-merge GHA workflow (on push to main) using
anthropics/claude-code-action for security review. Checks
for privilege escalation, secrets handling, security context
changes, hardcoded credentials. Opens a GitHub issue or
follow-up PR for findings. Provide CLAUDE.md guidance so
developers can run the same review locally before pushing.

## 24. Add AI RBAC review automation

Add a post-merge GHA workflow (on push to main) using
anthropics/claude-code-action for RBAC analysis. Path-filtered
to trigger on RBAC-related changes (config/rbac/, *_types.go,
controller*.go). Verifies least-privilege against controller
source code. Opens an issue for over-permissive RBAC findings.

## 25. Add AI release notes automation

Add a post-merge GHA workflow (on push to main) using
anthropics/claude-code-action to generate release note text
for user-facing changes (API/CRD changes, new features,
breaking changes). Opens a PR adding the suggested text to
the release notes file.

## 26. Add stale issue/PR workflow

Add a GHA workflow labeling stale issues after 120 days and PRs
after 14 days. Auto-close after 7 days with stale label.
Exempt labels: confirmed, security, good first issue.

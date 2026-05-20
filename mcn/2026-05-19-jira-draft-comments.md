---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [jira, draft-comments, ci, release, testing]
---

# Draft Jira Comments — For Review Before Posting

These are draft comments for three MCN Jira issues based on
tooling research across 50 K8s/CNCF project deep dives and 60+
tool health audits. See full details in the notes-ai/mcn/ notes
collection.

## CORENET-7086 — Enable Pre-merge testing automation

Draft comment:

---

Completed tooling research for this story. Based on deep dives
into 50 K8s/CNCF projects (Cilium, Istio, cert-manager, etcd,
Argo CD, Gateway API, etc.) and health audits of 60+ tools, here
is the proposed pre-merge CI stack.

### GitHub Actions Linting Workflow (18+ parallel jobs)

**Go linting** (via golangci-lint v2):

- 12 linters enabled: golangci-lint v2 with importas, modernize,
  funcorder, recvcheck, iface (identical only), depguard,
  forbidigo, promlinter, exhaustive, goheader
- kube-api-linter (KAL) for CRD API convention enforcement
  (separate custom golangci-lint binary via .custom-gcl.yml)
- gosec for security (bundled in golangci-lint)

**Non-Go linting:**

- markdownlint-cli2 (Markdown)
- yamllint (YAML)
- shellcheck + shfmt (shell scripts)
- hadolint (Dockerfiles)
- actionlint (GHA workflow correctness)
- zizmor (GHA security — 24 rules)
- conform or PR title check (commit message / Conventional Commits)
- kubeconform (K8s manifest schema validation)
- kube-linter (K8s manifest security — 40+ checks)
- lychee (link checking, replaces deprecated markdown-link-check)

**Security scanning:**

- govulncheck with SARIF output
- CodeQL variant analysis
- OSSF Scorecard (weekly + push)
- dependency-review-action (block PRs with known-vulnerable deps)
- SHA-pinned actions check
- Gitleaks/Betterleaks (secrets scanning in PR diffs)

**Testing:**

- Ginkgo/Gomega + envtest for unit/integration tests
- `-shuffle=on` and `go mod tidy -diff` flags
- go-test-coverage for per-package coverage ratcheting

**AI-powered PR review (3 non-blocking workflows):**

- Security review (RBAC, privilege escalation, secrets)
- RBAC change review (path-filtered, detailed permission analysis)
- Release notes suggestion (detects API/CRD changes)
- All use anthropics/claude-code-action with ANTHROPIC_API_KEY
  secret, read-only tools, sticky comments

**CRD validation:**

- controller-gen CI diff check (verify generated CRDs match
  committed)
- crdify for CRD breaking change detection
- go-apidiff for Go API backward compatibility

**CI workflow patterns:**

- dorny/paths-filter to skip jobs based on changed files
- Draft-aware test matrix (minimal CI for draft PRs)
- Composite result pattern for matrix status aggregation
- Workflow failure issue tracker (auto-create issues on CI failure)

### Prow / OpenShift CI (separate from this story — CORENET-7083)

GHA handles linting, unit tests, security scanning, AI reviews.
Prow handles real cloud E2E, upgrade testing, scale testing.
Follows the ovn-kubernetes pattern (dual CI).

### Suggested Subtasks

This story is too large for a single PR. Recommend splitting into:

1. Linting config files (.golangci.yml, .markdownlint.yml, etc.)
2. GHA linting workflow (18+ parallel jobs)
3. GHA unit test workflow
4. GHA branch enforcement + stale management
5. GHA periodic checks (weekly link check)
6. AI review workflows (3 files)
7. KAL integration (.custom-gcl.yml)
8. Security scanning workflows
9. Dependabot config
10. CRD validation CI

Full tool details, health audits, and pro/con analysis:
<https://github.com/dfarrell07/notes-ai/tree/main/mcn>

---

## CORENET-7089 — Release Automation and Versioning

Draft comment:

---

Completed tooling research for this story. Currently has no
description — proposing the following based on research across
50 K8s/CNCF projects.

### Proposed Scope

**Commit and changelog enforcement:**

- Conventional Commits format enforced via siderolabs/conform (Go
  binary, includes DCO + GPG validation) or PR title validation
  via action-semantic-pull-request (lighter weight, pairs with
  release-please)
- Per-PR changelog files (Contour pattern): every PR must include
  `changelogs/unreleased/{PR#}-{author}-{category}.md`. CI
  validates file exists and category matches release-note label.
  At release time, files are assembled into CHANGELOG

**Automated versioning:**

- release-please (GitHub Action, not App — App was shut down Aug
  2025) parses Conventional Commits to auto-bump versions and
  create release PRs with CHANGELOG updates

**Binary builds:**

- GoReleaser for cross-platform binary builds (if MCN ships a CLI)
- GoReleaser dry-run on config changes to catch release config bugs
- Fake release smoke test (daily `v9.9.9-fake` build) to catch
  release tooling rot

**Dependency management:**

- Dependabot for GHA monthly + Go modules weekly
- Dependabot auto-fix (regenerate generated code on dep updates)

**Backport automation:**

- korthout/backport-action: auto cherry-pick PRs to release
  branches based on `backport/release-X.Y` labels

**Supply chain security (at release time):**

- Cosign keyless signing for container images (OIDC via GitHub)
- Syft SBOM generation (SPDX format) attached to images
- SLSA Level 3 provenance attestations per image
- Release artifact verification (diff release vs source)

**Multi-branch maintenance:**

- Trivy multi-branch scanning (weekly across all release branches)
- Grype in parallel for defense in depth

### Subtasks for CORENET-7089

1. Conventional Commits setup (conform or PR title check)
2. release-please workflow
3. Per-PR changelog system (CI enforcement + release assembly)
4. Supply chain security (Cosign + Syft + SLSA)
5. Backport automation (label-driven cherry-pick)

### Key Architectural Decisions

- ko for upstream image builds (no Dockerfile), Konflux
  Dockerfiles for downstream
- release-please for versioning (not manual tagging)
- Per-PR changelog files (Contour pattern) — best pattern found
  across 50 projects
- Gitleaks/Betterleaks for secrets scanning (MIT, replaces
  TruffleHog AGPL)

Full tool details and health audits:
<https://github.com/dfarrell07/notes-ai/tree/main/mcn>

---

## CORENET-7085 — Enable upstream tests in Downstream CI

Draft comment:

---

Completed tooling research for this story. Currently has no
description — proposing the following.

### Scope for CORENET-7085

**Upstream test reuse in downstream Prow jobs:**

- Run upstream unit tests (`make test` or `go test ./...`) as a
  Prow presubmit job on the downstream openshift/mcn repo
- Run upstream E2E tests (Ginkgo + KIND) as Prow periodic or
  postsubmit jobs
- Use the same test flags as upstream: `-shuffle=on`, `-race`,
  `go mod tidy -diff`

**Coverage tracking:**

- Codecov integration for PR-level coverage comments
- go-test-coverage (vladopajic/go-test-coverage) for per-package
  coverage ratcheting — coverage can only go up

**Test quality (periodic):**

- go-ordered-test (weekly): detect tests with global state deps
- go-stress-test (nightly): detect flaky tests by running 1000x

**Downstream-specific testing:**

- Verify downstream build (UBI9 base, FIPS compliance) produces
  a working operator
- Run upstream conformance tests against downstream-built images
- Verify CRD compatibility between upstream and downstream builds

### Dependencies

- Depends on CORENET-7082 (downstream repo exists)
- Depends on CORENET-7086 (upstream tests exist to reuse)
- Depends on CORENET-7083 (Prow jobs configured)

Full tool details:
<https://github.com/dfarrell07/notes-ai/tree/main/mcn>

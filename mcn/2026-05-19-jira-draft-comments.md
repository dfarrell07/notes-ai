---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [jira, draft-comments, ci, release, testing]
---

# Draft Jira Comments — For Review Before Posting

Draft comments for three MCN Jira issues. Tone: sharing research
and opening discussion, not presenting final decisions.

## CORENET-7086 — Enable Pre-merge testing automation

Draft comment:

---

I've been doing some research into what CI tooling we should
consider for MCN. Looked at how ~50 K8s/CNCF projects handle
pre-merge testing (Cilium, Istio, cert-manager, etcd, Argo CD,
Gateway API, OVN-K, etc.) and wanted to share what I found as a
starting point for discussion.

### What other projects typically run in GHA

Most projects in this space use a combination of these tools.
Not saying we need all of them from day one, but this is the
landscape:

**Go linting** — golangci-lint v2 is the universal choice. Some
interesting newer linters worth considering:

- kube-api-linter (KAL) from kubernetes-sigs — enforces K8s API
  conventions on CRD types. Gateway API, openshift/api, and
  cluster-api use it. Pre-release but backed by SIG API Machinery.
  Thoughts on whether this is worth the setup complexity?
- importas — enforces consistent K8s import aliases (corev1 etc.)
- modernize — official Go team tool for modern idioms

**Non-Go linting** — the usual suspects that most projects run:
markdownlint, yamllint, shellcheck, hadolint. Two newer ones
worth discussing:

- kubeconform (replaces the abandoned kubeval) for K8s manifest
  validation
- kube-linter (Red Hat/StackRox) for K8s security checks — or
  possibly Kubescape (CNCF Incubating, broader coverage). Anyone
  have experience with either?
- actionlint + zizmor for GHA workflow linting/security — these
  are complementary, most projects run both

**Security scanning** — govulncheck seems like a no-brainer
(official Go team). CodeQL and OSSF Scorecard are standard. For
secrets scanning, Gitleaks (MIT) seems preferable to TruffleHog
(AGPL). Open to other suggestions here.

**AI-powered PR review** — this is more experimental. Some
projects (Chaos Mesh, Loki, Envoy Gateway) are using Claude or
Gemini to auto-review PRs for security issues, RBAC changes, and
release note suggestions. Non-blocking, informational only.
Interested in whether the team thinks this is worth trying.

**Testing** — Ginkgo/Gomega + envtest seems like the standard
choice for K8s operators. One interesting pattern: using
`-shuffle=on` to catch ordering-dependent tests from day one.

### GHA vs Prow split

The pattern from OVN-K and other OpenShift operators is GHA for
linting/unit/images and Prow for real cloud E2E. That seems right
for MCN too — CORENET-7083 covers the Prow side.

### This story might be too big

Looking at the scope, this could be 10+ separate PRs. Might be
worth creating subtasks. Some natural groupings:

1. Linting config files
2. GHA linting workflow
3. GHA unit test + branch enforcement workflows
4. Security scanning workflows
5. AI review workflows (if we want them)
6. Dependabot config

I've got detailed notes on all of these tools including health
audits, license checks, and alternatives analysis. Happy to share
more details on any specific tool or walk through the research.

---

## CORENET-7089 — Release Automation and Versioning

Draft comment:

---

This story doesn't have a description yet. I've been looking at
how other K8s projects handle release automation and have some
options to discuss.

### Commit message format

Two main approaches I've seen:

- **Conventional Commits** (feat:, fix:, etc.) — enables automated
  changelog generation. Most projects going this route use either
  siderolabs/conform (Go binary, also does DCO checks) or PR title
  validation with action-semantic-pull-request
- **Free-form** with per-PR changelog files (Contour pattern) —
  every PR includes a small changelog file, assembled at release
  time. Avoids commit message enforcement but still gets structured
  release notes

I'm leaning toward Conventional Commits + per-PR changelog files
(some projects do both) but curious what others think.

### Versioning automation

release-please (Google) seems to be the most popular option for
automating version bumps from Conventional Commits. Note: the
GitHub App was shut down in Aug 2025 — must use the Action.
Alternative is manual tagging, which is simpler but more
error-prone.

### Other release tooling to consider

- Dependabot for automated dependency updates (GHA monthly, Go
  modules weekly)
- korthout/backport-action for auto cherry-picks to release
  branches
- Cosign for image signing, Syft for SBOMs, SLSA for provenance
  (these could be Phase 3 rather than day one)
- GoReleaser dry-run on config changes — catches release script
  bugs before release day

### Questions for the team

- Do we want Conventional Commits from day one, or start simpler?
- Per-PR changelog files — worth the overhead?
- When should we add supply chain security (Cosign/SLSA)? Day one
  or closer to first release?

More details in my research notes if anyone wants to dig deeper.

---

## CORENET-7085 — Enable upstream tests in Downstream CI

Draft comment:

---

This story doesn't have a description yet. Here are some initial
thoughts based on how other OpenShift operators handle this.

### What I think this should cover

- Running upstream unit tests as a Prow presubmit on the downstream
  repo (openshift/mcn)
- Running upstream E2E tests against downstream-built images
- Coverage tracking (Codecov seems standard, with go-test-coverage
  for per-package ratcheting)
- Periodic test quality checks (detecting flaky tests, test
  isolation issues)

### Downstream-specific testing

Beyond just rerunning upstream tests, we probably need:

- Verify the downstream build (UBI9, FIPS) produces a working
  operator
- CRD compatibility between upstream and downstream

### Dependencies

This is blocked on several other stories:

- CORENET-7082 (downstream repo needs to exist)
- CORENET-7086 (upstream tests need to exist first)
- CORENET-7083 (Prow jobs need to be configured)

Would be good to confirm the sequencing here. Thoughts?

---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [jira, draft-comments, ci, release, testing]
---

# Draft Jira Comments — For Review Before Posting

Draft comments for three MCN Jira issues. Sharing research and
direction, open to input but not blocking on it.

## CORENET-7086 — Enable Pre-merge testing automation

Draft comment:

---

I've been researching CI tooling for MCN — looked at how ~50
K8s/CNCF projects handle pre-merge testing (Cilium, Istio,
cert-manager, etcd, Argo CD, Gateway API, OVN-K, etc.). Here's
what I'm planning and some of the options I'm considering.

### GHA linting workflow

**Go linting** — golangci-lint v2 with a strict config. Planning
to start from the Submariner config and add some newer linters:

- kube-api-linter (KAL) from kubernetes-sigs for CRD API
  conventions. Pre-release but backed by SIG API Machinery and
  already used by Gateway API, openshift/api, and cluster-api.
  Adds some setup complexity (custom golangci-lint binary) but
  catches things nothing else does.
- importas for consistent K8s import aliases
- modernize (official Go team) for modern idioms
- depguard/forbidigo for blocking deprecated packages and
  functions

**Non-Go linting** — standard tools: markdownlint, yamllint,
shellcheck, hadolint. Plus:

- kubeconform for K8s manifest validation (replaces abandoned
  kubeval)
- kube-linter (Red Hat/StackRox) for K8s security checks.
  Kubescape is another option here with broader coverage.
- actionlint + zizmor for GHA workflow correctness and security

**Security scanning** — govulncheck (official Go team), CodeQL,
OSSF Scorecard, dependency-review-action, Gitleaks for secrets.

**AI-powered PR review** — planning to try Claude-based
non-blocking reviews for security issues, RBAC changes, and
release note suggestions. Several projects are doing this now
(Chaos Mesh, Loki, Envoy Gateway). Easy to add or remove since
they're informational only.

**Testing** — Ginkgo/Gomega + envtest, `-shuffle=on` from day
one.

### GHA vs Prow

Planning to follow the OVN-K pattern: GHA for linting/unit/images
and Prow for real cloud E2E (covered by CORENET-7083).

### Splitting this story

This is a lot of scope for one story. I'll likely split it into
subtasks as I work through it — linting configs, GHA workflows,
security scanning, etc.

I have detailed notes on all the tools including health/license
audits. Happy to share if anyone wants to dig into specific
choices.

---

## CORENET-7089 — Release Automation and Versioning

Draft comment:

---

This story needs a description. Here's what I'm thinking based
on research into how other K8s projects handle releases.

### Direction I'm leaning

I'm reading this story as covering the **upstream** release
process — how we version, tag, and publish releases from the
upstream repo. Downstream Konflux pipelines (stage/prod releases,
image signing, FBC catalogs) are a separate concern covered by
the downstream stories (7079, 7082, 7084).

**Commit format** — Conventional Commits (feat:, fix:, etc.).
This enables automated changelog and version bumping. Planning
to use siderolabs/conform (Go binary, also handles DCO) or PR
title validation for enforcement.

**Changelog** — per-PR changelog files (the Contour pattern).
Every PR adds a small file in `changelogs/unreleased/`, CI
enforces it, files get assembled at release time. This was the
best release note pattern I found across 50 projects — avoids
the single-CHANGELOG merge conflict problem.

**Versioning** — release-please (GitHub Action) for automated
version bumps from Conventional Commits. Note: the GitHub App
was shut down Aug 2025, must use the Action.

**Dependency updates** — Dependabot for GHA (monthly) and Go
modules (weekly). Auto-fix workflow to regenerate code on dep
updates.

**Backports** — korthout/backport-action for auto cherry-picks
based on labels.

Open to different approaches on any of this.

---

## CORENET-7085 — Enable upstream tests in Downstream CI

Draft comment:

---

This story needs a description. Here's what I think it should
cover based on the OVN-K and other OpenShift operator patterns.

### Proposed scope

- Run upstream unit tests as a Prow presubmit on the downstream
  repo (openshift/mcn)
- Run upstream E2E tests against downstream-built images
- Coverage tracking via Codecov with per-package ratcheting
- Periodic test quality checks (flaky test detection, test
  isolation)
- Downstream-specific verification: UBI9/FIPS build produces
  a working operator, CRD compatibility between upstream and
  downstream

### Dependencies

Blocked on CORENET-7082 (downstream repo), CORENET-7086 (upstream
tests exist), and CORENET-7083 (Prow configured). Will start
planning details once those are further along.

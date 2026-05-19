---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, automation, ecosystem, devex, supply-chain, testing]
---

# K8s Ecosystem CI Innovations for MCN

Novel CI and developer experience tooling discovered from surveying
Cilium, Calico, Istio, Gateway API, Kubernetes, cert-manager,
Crossplane, and the broader CNCF ecosystem. Only patterns NOT
already covered in our Submariner or OpenShift surveys.

## Strongly Recommended — Adopt for MCN

### ko — Replace Dockerfiles for Go Image Builds

CNCF Sandbox project. Builds Go applications into container images
without a Dockerfile or Docker daemon. Images are minimal (<10MB),
reproducible, and multi-platform by default. Built-in SBOM
generation. Used by Knative, Tekton, and many Go K8s projects.

```bash
KO_DOCKER_REPO=quay.io/mcn ko build ./cmd/manager
```

Integrates with `ko apply` for direct deployment to a cluster and
Skaffold as a builder option. Supports `--watch` mode for
live-reload development.

**Why for MCN**: Eliminates upstream Dockerfiles entirely. Faster
builds, automatic SBOMs, reproducible by default. Still need
Konflux Dockerfiles for downstream (UBI9 base requirement).

### release-please — Automated Versioning and Changelogs

Google's tool. Parses Conventional Commit messages (`fix:`, `feat:`,
`feat!:`) to determine version bumps. Automatically creates a
"Release PR" with updated CHANGELOG.md and version bumps. Merging
the Release PR triggers tag creation and GitHub Release publishing.

Pair with Conventional Commits enforcement via Lefthook pre-commit
hooks or commitlint in CI. Strongly recommends squash-merge
workflow for clean linear history.

**Why for MCN**: Automates version bumps, changelog generation, and
GitHub Releases with minimal configuration. Replaces manual
release note writing for upstream releases.

### Lefthook — Pre-Commit Hooks (Go Native)

Written in Go, single binary, no Python dependency. Supports
parallel hook execution (pre-commit.com does not). Typical hooks:
golangci-lint, go vet, go mod tidy, controller-gen checks.

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    lint:
      run: golangci-lint run --new
    mod-tidy:
      run: go mod tidy && git diff --exit-code go.sum
    generate:
      run: make manifests && git diff --exit-code config/crd/
```

**Why for MCN**: Catches issues before they enter CI. Every
developer gets consistent pre-commit checks.

### kube-api-linter (KAL) — CRD API Convention Enforcement

(Also found in content-type-tooling notes, confirmed here as used
by Gateway API.) Enforces K8s API Conventions mechanically on CRD
Go types. Catches: booleans that should be enums, missing omitempty,
Phase anti-pattern, wrong json tag casing, incorrect Conditions
patterns.

**Why for MCN**: Automates the mechanical parts of K8s API review.
Catches what human reviewers flag. OpenShift API repo uses it.

### dorny/paths-filter — Conditional CI Execution

Filters CI jobs based on which files changed in a PR. Skip
expensive tests when only docs changed, skip doc builds when only
Go code changed. Used extensively by Cilium.

```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    filters: |
      go:
        - '**.go'
        - 'go.mod'
      docs:
        - 'docs/**'
        - '**.md'
```

**Why for MCN**: Reduces CI time and cost from day one.

### go-ordered-test — Test Isolation Detector (Istio)

Runs each test individually in isolation, catching tests that
depend on global state set by other tests. Simple shell script
wrapper: `go test -exec ./tools/go-ordered-test ./...`

**Why for MCN**: Finds "pass locally, fail in CI" bugs caused by
test coupling. Run weekly in periodic CI.

### go-stress-test — Flaky Test Detector (Istio)

Runs each test in a loop (default: 1000 times or 10 seconds) to
identify flaky tests. `go test -exec ./tools/go-stress-test ./...`

**Why for MCN**: Proactive flaky test detection rather than
relying on CI reruns. Run nightly against new test files.

### Workflow Telemetry Action (Cilium)

`cilium/workflow-telemetry-action` — collects timing metrics for
every GitHub Actions job and step. Generates Gantt-style trace
visualizations posted as PR comments and job summaries.

**Why for MCN**: Zero-effort CI observability. Makes CI
optimization data-driven. Add one step at workflow start.

### Network Policy Conformance Tests

`kubernetes-sigs/network-policy-api` conformance suite. Tests for
AdminNetworkPolicy and BaselineAdminNetworkPolicy. If MCN
implements these, passing conformance tests demonstrates
interoperability and positions for upstream recognition.

**Why for MCN**: Standardized conformance suite for networking
operators. Run in CI against MCN's implementation.

### PR Checklist Enforcement (Crossplane)

`mheap/require-checklist-action` — blocks PR merge unless PR
description contains a checklist and all items are checked.

**Why for MCN**: Simple process enforcement ensuring PR authors
complete self-review steps. Low effort, high value.

### Backport Action (Crossplane, Cilium)

`korthout/backport-action` — automatically creates backport PRs
when a PR with `backport/release-X.Y` labels is merged. Supports
merge commits and multi-commit PRs.

**Why for MCN**: Eliminates manual cherry-pick work once release
branches exist.

## Recommended — Add Before First Release

### Cosign + Syft — Supply Chain Security

Sign container images with cosign keyless signing (OIDC, no key
management). Generate SBOMs with Syft. Attach SBOM attestations
to images. Upload SBOMs as release assets.

Three-layer approach:

1. Sign images with cosign keyless signing in CI
2. Generate SBOMs with Syft, attach via `cosign attach sbom`
3. Generate SLSA provenance via `actions/attest-build-provenance`

Used by Cilium, cert-manager, ECK.

### OSSF Scorecard (cert-manager)

Weekly supply chain security assessment. Uploads SARIF to GitHub
code scanning. Generates public scorecard badge. Single workflow
file.

### elastic/crd-ref-docs — API Documentation Generation

Scans Go source trees for CRD types, generates Markdown or
AsciiDoc API reference. Used by Elastic (ECK operator).
Integrates into `make generate` workflow.

Alternative: `kubespec.dev` for interactive YAML-centric docs
once the API stabilizes.

### CRD CEL Validation Testing Across K8s Versions (Gateway API)

Tests CRD CEL validation expressions against a matrix of
Kubernetes versions. Uses envtest to spin up API servers at each
version. Catches version-specific CEL engine differences.

### overcover — Per-Package Coverage Thresholds (Istio)

Enforces per-package code coverage thresholds. Prevents "high
overall coverage masks uncovered packages" problem.

### Race Detection Conformance Runs (Cilium)

Dedicated nightly workflow running full test suite with Go's
`-race` detector. Separate from normal CI due to overhead.

### Release Artifact Verification (Gateway API)

On release publish, builds install YAML from tagged source and
diffs against published release asset. Ensures release artifacts
match source.

### Renovate over Dependabot

More flexible: supports grouping updates, scheduling, custom PR
templates, auto-merge policies. Groups K8s dependencies together
(controller-runtime, client-go, apimachinery must move in
lockstep). Single PR can update across go.mod, Dockerfile, and
CI configs.

## Nice-to-Have — Consider as Project Matures

### Connectivity Disruption Testing (Cilium)

Measures connection disruption during upgrades, configuration
changes, and failovers. Tests L4 and L7 traffic disruption with
configurable concurrency. Most projects test "does it work after
upgrade" but not "how much traffic was disrupted during upgrade."

### benchstat Benchmark Regression Detection (Istio)

Stores Go benchmark results in object storage, compares with
`benchstat` on PR. Reports performance regressions automatically.

### devcontainer.json

VS Code Remote Containers / GitHub Codespaces standard. Ensures
every developer has identical toolchain (Go, controller-gen, ko,
kind, etc.).

### Comment-Driven Test Orchestration (Cilium Ariane)

GitHub App that watches PR comments for trigger phrases
(`/test`, `/ci-gateway-api`) and dispatches workflows with
dependency resolution. Replaces Prow while keeping full control
in GitHub Actions.

### Feature Summary Report (Cilium)

Daily workflow generating a report of features enabled/tested
across the test matrix. Prevents feature test coverage from
silently degrading.

### Nightly testlinter (Istio)

Custom linter enforcing: skipped tests must reference a GitHub
issue, e2e tests must check `testing.Short()`, integration tests
follow naming conventions.

### octo-sts — Short-Lived GitHub Tokens (cert-manager)

GitHub App issuing short-lived tokens scoped to specific repos
and permissions. Eliminates long-lived PATs in CI workflows.

## Updated MCN CI Job Count

With all discoveries, the full linting workflow grows from 18
jobs (content-type-tooling notes) to potentially 22+:

1-18. (From content-type-tooling notes — golangci-lint with KAL,
kubeconform, kube-linter, hadolint, actionlint, zizmor,
go-apidiff, license headers, etc.)

New additions:

1. `paths-filter` — conditional job execution (meta-job that
    gates others)
2. `pr-checklist` — verify PR description checklist complete
3. `cel-validation` — test CRD CEL expressions across K8s
    versions
4. `workflow-telemetry` — CI timing metrics (meta-step, not a
    separate job)

Periodic/nightly additions (not on every PR):

- `go-ordered-test` — weekly test isolation check
- `go-stress-test` — nightly flaky test detection
- `race-conformance` — nightly race detection run
- `ossf-scorecard` — weekly security assessment
- `network-policy-conformance` — conformance suite run

## Key Decisions for MCN

### ko vs Dockerfile for upstream

ko eliminates Dockerfiles entirely for Go projects. But:

- Downstream (Konflux) still needs UBI9-based Dockerfiles
- ko images use distroless/static by default (good)
- ko builds are reproducible and include SBOMs automatically

Recommendation: Use ko for upstream, Konflux Dockerfiles for
downstream. This is a departure from Submariner's pattern (which
uses Dockerfiles for both) but aligns with modern CNCF practice.

### release-please vs manual releases

release-please automates the upstream release cycle entirely:

- Conventional Commits drive version bumps
- CHANGELOG.md generated automatically
- GitHub Releases created on merge of Release PR

This replaces the need for a manual release workflow for upstream.
Downstream (Konflux) releases remain a separate 20-step process.

### Renovate vs Dependabot

Renovate is more flexible (grouping, auto-merge policies, K8s
dependency lockstep). But Dependabot is zero-setup on GitHub.

Recommendation: Start with Dependabot (zero effort). Switch to
Renovate if dependency management gets complex (multiple modules,
cross-file updates needed).

## Extended Survey — 35 Additional Projects

Surveyed 35 more K8s projects (total: 52). Below are the best
novel patterns organized by category.

### CI Security Hardening

**step-security/harden-runner** (Argo CD, External Secrets) —
network egress control for GHA runners. Block mode with explicit
endpoint allowlists for workflows handling secrets or publishing
artifacts. Strong supply chain security primitive.

**zgosalvez/github-actions-ensure-sha-pinned-actions** (Kyverno) —
verifies every GHA reference uses SHA pin, not mutable tag.
Prevents tag hijacking attacks. Supports allowlists for actions
requiring semver tags.

**TruffleHog secret scanning** (Loki) — scans PR diffs for
verified secrets with `--only-verified` flag. Minimal false
positives, complements GitHub's built-in secret scanning.

**actions/dependency-review-action** (External Secrets, Tekton) —
GitHub-native, blocks PRs introducing known-vulnerable packages.
Free, should be enabled on every project.

### CI Workflow Management

**Workflow failure issue tracker** (Kyverno) — when CI on main
fails, auto-creates GitHub issue with run metadata. When it
later succeeds, auto-closes the issue. Much better than email
notifications.

**Pass-CI / Skip-CI companion pattern** (Harbor, Chaos Mesh) —
solves GitHub's "required but skipped checks block merge"
problem. Companion workflow provides no-op jobs with same names
for paths the real CI skips.

**Fake DCO for merge queue** (Jaeger, Chaos Mesh) — stub workflow
on `merge_group` events for checks that only run on
`pull_request`. Unblocks merge queue.

**Trust-tiered CI parallelism** (Jaeger) — trusted contributors
(org members, 5+ merged PRs) get parallel CI (~10 min).
External contributors get sequential (~30 min). Saves compute
while gatekeeping.

**Draft-PR-aware reduced test matrix** (Tekton) — draft PRs run
minimal matrix (amd64, latest K8s). Non-draft PRs run full
matrix (multi-arch, multi-version, all feature gates).

### Testing Innovations

**Antithesis autonomous fault injection** (etcd) — pushes images
to Antithesis cloud platform for random fault injection (network
partitions, crashes, clock skew). Finds correctness bugs in
distributed systems that scripted e2e misses.

**CIFuzz / OSS-Fuzz in PR CI** (CoreDNS, MetalLB, KubeEdge) —
Google's continuous fuzzing infrastructure in every PR. 10-20
min of fuzzing per PR. Catches security-relevant input-handling
bugs before merge. Register project with OSS-Fuzz.

**Version skew testing** (Dapr) — tests N-1 compatibility between
control plane and sidecar. Deploys both current and latest
release, exercises them together. Critical for operators with
upgrade semantics.

**Binary resource regression** (Dapr) — numpy/scipy-based
comparison of binary size and virtual memory between PR and
baseline. Enforces 7MB delta limit on binary growth.

**Downstream consumer testing** (OPA) — checks out downstream
projects, replaces their dependency via `go mod edit -replace`,
runs their tests. Catches breaking changes before release.

**k6 performance testing** (Kyverno) — Grafana k6 load testing
against real KinD cluster with Prometheus. Tests baseline K8s
vs operator deployed vs operator + policies. Catches perf
regressions.

**Automated flaky test detection with auto-filed issues** (etcd) —
scrapes CI run history, detects intermittent failures, auto-
creates GitHub issues for flaky tests.

**13-provider E2E matrix** (Flagger) — tests progressive delivery
across 13 service mesh/ingress providers in one workflow.

**OCI Distribution conformance testing** (Harbor) — formal spec
compliance tests, not just functional e2e.

### Release and Changelog

**Changelog file enforcement** (Contour) — every PR must include
a file `changelogs/unreleased/PR#-author-category`. Category
must match PR's release-note label. Produces clean release
notes automatically.

**Changelog-driven release trigger** (Cluster API) — merging a
changelog file IS the release trigger. No manual tag pushing.
Workflow extracts version from filename, creates branch and
tag.

**Weekly release build smoke test** (Cluster API) — daily fake
release builds catch tooling regressions continuously rather
than at release time.

**CVE container rebuild without re-release** (Strimzi) — rebuild
and re-publish release containers with version suffix when CVEs
found. Patches base images without new source release.

**Monthly release issue auto-creation** (Envoy Gateway) — reads
supported versions from config, creates templated release
tracking issues per active branch on the 1st of every month.

**CRD JSON Schema as release artifact** (Flux) — generates
OpenAPI JSON schemas from CRDs at release time. Enables IDE
validation of custom resources.

### Developer Experience

**PR quota manager** (Jaeger) — limits concurrent open PRs per
contributor based on merge history. New contributors get 1
slot, scaling up with merges. Prevents review queue flooding.

**Auto-reviewer assignment by file path** (KEDA) — assigns
reviewers based on which files changed via glob patterns.
Ensures domain experts see relevant changes.

**Merge conflict label bot** (Kyverno) — auto-labels PRs with
merge conflicts, removes label when resolved.

**Stale branch cleanup** (Kyverno) — daily cron deletes branches
with no commits in 7 days, respecting prefix allowlists.

**PR branch auto-updater** (Kyverno) — auto-rebases all open PR
branches when main is pushed to.

**Workflow auto-approval for returning contributors** (Volcano) —
auto-approves pending workflow runs for non-first-time
contributors.

**SSH debug auto-enable on CI re-run** (Rook) — if a test fails
and you re-run it, SSH access auto-enables. Removes friction
from failure investigation.

**Sprint lifecycle automation** (Longhorn) — sprint rotation,
testing item Slack notifications, review reminders via GHA +
Projects V2 GraphQL.

### Supply Chain

**SLSA Level 3 for binaries AND images** (Flux) — separate
provenance generation per registry. Gold standard.

**Cosign keyless signing with GitHub OIDC** (MetalLB, Flagger) —
signs image digests at publish time.

**Dual-location release asset verification** (etcd) — downloads
release binaries from both GitHub and GCS, checksums must
match.

**Third-party image mirroring** (Dapr) — monthly mirror of
external Docker Hub images to project-owned GHCR under
`3rdparty/` namespace.

**Go dependency submission API** (Telepresence) — submits Go
dependency graph to GitHub for Dependabot alerts.

### Miscellaneous

**Helm chart diff CI** (Loki) — renders charts with multiple
value-file scenarios, runs `helm diff` to show exactly what
K8s resources would change. Posts diff for review.

**Inclusive language linting** (Tekton) — `get-woke/woke-action`
on changed files.

**Merge conflict marker detection** (Chaos Mesh) — scans for
unresolved `<<<<<<<` markers in committed files.

**Engine version bump soft-block** (Falco) — detects when source
changes but version header wasn't bumped. Posts reminder and
applies `/hold`.

**Dependabot config auto-regeneration** (Tekton) — weekly
workflow regenerates `.github/dependabot.yml` from config
template, auto-creates PR if changed.

**Dependabot PR auto-fix** (Cluster API) — when Dependabot opens
a PR, runs `make generate` and auto-commits updated generated
code.

## Full Project List (52 surveyed)

1-5: Submariner repos (operator, shipyard, submariner,
lighthouse, admiral)
6-10: OVN-Kubernetes, CNO, CNCC, multus-cni, network-tools
11-17: cluster-ingress-operator, MCO, CKAO, controller-runtime,
operator-sdk, Cilium, Calico
18-22: Istio, Gateway API, cert-manager, Crossplane, Argo CD
23-27: Flux, Kyverno, Prometheus Operator, External Secrets,
Knative
28-32: Tekton, Velero, Cluster API, MetalLB, Contour
33-37: Linkerd, CoreDNS, etcd, KEDA, Rook
38-42: Thanos, Loki, Jaeger, OPA, Envoy Gateway
43-47: KubeVirt, Strimzi, Keptn, Falco, Dapr
48-52: Harbor, Longhorn, Karmada, Volcano, Flagger, Chaos Mesh,
Telepresence, Emissary-Ingress, KubeEdge

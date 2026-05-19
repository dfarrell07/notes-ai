---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, prow, gha, openshift, ovn-kubernetes, cno, patterns]
---

# OpenShift CI Patterns — Survey for MCN Planning

Survey of CI across OpenShift networking and operator repos. Covers
the GHA vs Prow split, full job inventories, and novel patterns MCN
should consider adopting.

## Repos Surveyed

- **ovn-kubernetes** (ovn-org + openshift fork) — closest MCN analog
- **cluster-network-operator** (CNO) — main OpenShift network operator
- **cloud-network-config-controller** (CNCC) — cloud EgressIP controller
- **multus-cni** — meta-CNI for multiple network attachments
- **network-tools** — debug/diagnostic tools image
- **cluster-ingress-operator** — networking-adjacent operator
- **machine-config-operator** (MCO) — large, mature operator
- **cluster-kube-apiserver-operator** (CKAO) — core operator
- **controller-runtime** (upstream) — controller framework
- **operator-sdk** (upstream) — operator tooling

## 1. GHA vs Prow Split — The Pattern

All OpenShift operators use Prow/ci-operator as their primary CI.
Some also use GitHub Actions for supplementary checks. The split:

### Prow-Only Repos (no GitHub Actions at all)

- cluster-network-operator
- cloud-network-config-controller
- network-tools
- cluster-ingress-operator
- machine-config-operator
- cluster-kube-apiserver-operator

### Dual CI (GHA + Prow)

- ovn-kubernetes — 8 GHA workflows + 25 Prow jobs + 61 ci-operator
  config files spanning OCP 4.6 through 5.1
- multus-cni — 9 GHA workflows + Prow for OCP e2e

### GHA-Only (upstream projects, no OpenShift downstream)

- controller-runtime — GHA + Prow (from kubernetes/test-infra)
- operator-sdk — pure GHA (9 workflows)

### What Goes Where

**GitHub Actions** handles:

- Linting (golangci-lint, markdownlint, yamllint, shellcheck)
- Unit tests with coverage
- Container image builds and pushes (ghcr.io)
- Performance benchmarks (on self-hosted runners)
- Documentation site builds
- License header checks
- PR labeling and stale management
- AI code review (CodeRabbit)
- Retest/cancel bot commands
- Supply chain security scoring (OSSF Scorecard)

**Prow/ci-operator** handles:

- Real cloud e2e (AWS, GCP, Azure, vSphere, OpenStack, IBM Cloud,
  bare metal)
- Upgrade tests (standard, IPsec, cross-version)
- Performance/scale tests (up to 1500 worker nodes)
- HyperShift and MicroShift conformance
- Security scanning (openshift-ci-security workflow)
- Dependency verification (go-verify-deps step)
- Windows node testing
- FIPS compliance testing
- Gateway mode migration tests
- Network CIDR expansion tests

**MCN recommendation**: Use both. GHA for linting, unit tests, image
builds, AI reviews. Prow for cloud e2e, upgrades, scale testing.
This matches the ovn-kubernetes pattern — the closest MCN analog.

## 2. Prow Configuration Details

### ci-operator Config Structure

Each repo has configs at:
`openshift/release/ci-operator/config/openshift/<repo>/`

Per-branch YAML files with variant suffixes:

- `openshift-<repo>-master.yaml` — main config
- `openshift-<repo>-master__periodics.yaml` — periodic-only jobs
- `openshift-<repo>-master__4.20-upgrade-from-stable-4.19.yaml` —
  upgrade variant
- `openshift-<repo>-master__okd-scos.yaml` — OKD variant

### Build Root

Each repo has `.ci-operator.yaml` at root specifying the build image:

```yaml
build_root_image:
  name: release
  namespace: openshift
  tag: rhel-9-release-golang-1.25-openshift-4.22
```

### Common Prow Test Types

Every OpenShift operator repo includes:

- `unit` — Go unit tests in container
- `verify` or `lint` — Go linting
- `verify-deps` — go.mod/vendor consistency (go-verify-deps step)
- `security` — openshift-ci-security workflow
- `e2e-aws-ovn` — basic AWS e2e
- `e2e-aws-ovn-upgrade` — upgrade testing

### Step Registry Workflows

E2e tests reference shared workflows from the step registry:

- `openshift-e2e-aws-ovn` — standard AWS e2e
- `openshift-upgrade-aws-ovn` — AWS upgrade
- `openshift-e2e-gcp-ovn` — GCP e2e
- `openshift-e2e-azure-ovn` — Azure e2e
- `openshift-e2e-vsphere-ovn` — vSphere e2e
- `baremetalds-e2e` — bare metal on Equinix
- `hypershift-aws-conformance` — HyperShift
- `openshift-ci-security` — security scanning
- `go-verify-deps` — dependency verification

### CI Resource Optimization Patterns

All repos use Prow's filtering to save CI resources:

- `skip_if_only_changed` — skip e2e for doc/metadata-only PRs
  (regex: `\.md$|^docs/|^OWNERS`)
- `run_if_changed` — only run domain-specific tests when relevant
  code changes (e.g., encryption tests only when encryption code
  changes in CKAO)
- `optional: true` + `always_run: false` — tests that only run
  when manually triggered or labeled

### Prow Merge Requirements (release branches)

Labels required: `approved`, `lgtm`, `verified`,
`jira/valid-bug`, `jira/valid-reference`,
`backport-risk-assessed`

Labels that block: `do-not-merge/hold`, `needs-rebase`,
`jira/invalid-bug`, `backports/unvalidated-commits`

## 3. OVN-Kubernetes — Deep Dive (Closest MCN Analog)

### GitHub Actions (8 workflows)

The main CI (`test.yml`, 912 lines) runs:

- golangci-lint v2.5.0
- Build base + PR container images
- Unit tests with Coveralls coverage
- 45+ e2e matrix jobs on KIND clusters testing combinations of:
  - Targets: shard-conformance, control-plane, multi-homing,
    multi-node-zones, BGP, EVPN, network-segmentation, serial,
    tools, and more
  - HA modes: HA, noHA
  - Gateway modes: local, shared
  - IP families: ipv4, ipv6, dualstack
  - Interconnect: disabled, single-node, multi-node zones
- Upgrade tests and dual-stack conversion tests

Other workflows: performance testing (kube-burner on 32-CPU
runners), multi-arch image builds (ghcr.io), docs (MkDocs),
license headers (Apache SkyWalking Eyes), PR auto-labeling,
stale management, /retest bot.

### Prow (25+ jobs on master)

Cloud e2e across AWS, GCP, Azure, vSphere, OpenStack, IBM Cloud,
Equinix bare metal. Upgrade tests across cloud providers. HyperShift
and MicroShift conformance. Gateway mode migration. Scale testing
up to 1500 workers. TechPreview variants. Windows node testing.

### Key OVN-K Patterns for MCN

- KIND as e2e backbone (GHA) + real clouds via Prow
- Custom /retest GitHub Action (reimplements Prow's /retest)
- CodeRabbit AI review (`.coderabbit.yml`, "chill" profile)
- PR auto-labeling by file path (`.github/labeler.yml`)
- Performance tests on self-hosted runners with kube-burner
- Coredump detection in test teardown
- No Dependabot or Renovate (manual dependency updates)

## 4. Novel CI Patterns MCN Should Consider

### High Priority — Adopt These

**go-apidiff for API compatibility** (from controller-runtime)

- Tool: `go-apidiff` v0.8.3
- Compares PR against base commit to detect breaking API changes
- Currently optional (warns but doesn't block) in controller-runtime
- Perfect for catching breaking CRD/API changes before merge
- MCN should adopt this for its CRD API types

**openshift-ci-security workflow** (all OpenShift repos)

- Standard security scanning Prow job
- Every OpenShift operator should have this
- MCN: add to Prow config from day one

**verify-deps step** (all OpenShift repos)

- `go-verify-deps` step registry reference
- Validates go.mod/vendor consistency
- MCN: add to Prow config from day one

**run_if_changed targeted testing** (from CKAO)

- Only run domain-specific tests when relevant code changes
- Example: encryption tests only when encryption code changes
- Saves significant CI resources on large test matrices

**skip_if_only_changed for e2e** (all OpenShift repos)

- Skip expensive cloud e2e for doc-only or metadata-only PRs
- Standard regex: `\.md$|^docs/|^OWNERS`
- MCN: add to all Prow e2e job definitions

**Upgrade testing** (all OpenShift repos)

- At minimum `openshift-upgrade-aws` workflow
- MCO also tests reverse upgrades (downgrade compatibility)

### Medium Priority — Consider These

**OSSF Scorecard** (from controller-runtime)

- Weekly supply chain security analysis via GHA
- Uploads SARIF to GitHub Code Scanning
- Publishes score to OpenSSF
- Low effort to set up, good visibility

**PR title verification** (from controller-runtime)

- Enforces conventional commit-style prefixes
- Helps with automatic changelog generation
- Categories: breaking, feature, bugfix, docs, release, infra

**CodeRabbit AI review** (from ovn-kubernetes,
cluster-ingress-operator)

- Automated AI code review on PRs
- "chill" profile minimizes noise
- Excludes vendor directory
- Alternative or complement to our Claude-based AI reviews

**AGENTS.md** (from cluster-ingress-operator)

- Comprehensive context document for AI coding agents
- System context, directory layout, controller patterns,
  constraints
- Helps AI agents make better suggestions
- MCN should create this alongside CLAUDE.md

**Changelog fragment system** (from operator-sdk)

- Structured per-PR changelog entries in `changelog/fragments/`
- Each PR that warrants a release note adds a fragment file
- Fragments compiled into changelog at release time

**Error/log message format checker** (from operator-sdk)

- `hack/check-error-log-msg-format.sh`
- Enforces Go style: lowercase errors, uppercase log messages
- Simple grep-based check, easy to adopt

**E2e test completeness checker** (from cluster-ingress-operator)

- `verify-e2e-test-all-presence.sh`
- Verifies every test function is called in TestAll
- Prevents orphaned tests that silently stop running

**gomodcheck upstream alignment** (from controller-runtime)

- Custom tool validating k8s.io module versions stay in sync
- Uses `.gomodcheck.yaml` to declare upstream refs
- Catches dependency version drift

**Third-party license verification** (from ovn-kubernetes)

- `make verify-third-party-licenses`
- Generates and verifies third-party license files
- Ensures license compliance is tracked

### Lower Priority — Interesting but Not Urgent

**Out-of-change (reverse) upgrade testing** (from MCO)

- Swaps initial and latest releases to test downgrade
- Unique and thorough but heavy CI cost

**FIPS compliance testing** (from MCO)

- `e2e-aws-ovn-fips` with `FIPS_ENABLED: "true"`
- Includes `fips-check` step in pre chain
- Needed if MCN handles cryptographic operations

**Cert rotation testing at time skews** (from CKAO)

- Tests cert rotation at 90d/180d/1y/2y/3y skews
- Both suspend and shutdown modes
- Relevant if MCN manages certificates

**Performance benchmarks** (from ovn-kubernetes)

- kube-burner workloads on self-hosted 32-CPU runners
- Prometheus metrics + Elasticsearch reporting
- Consider once MCN has performance-sensitive code paths

**Dependabot auto-fixup** (from controller-runtime)

- GHA workflow auto-runs `make modules` on dependabot PRs
- Auto-commits go.sum and vendor updates

**Snyk with documented CWE exclusions** (from MCO)

- `.snyk` policy file with justified exclusion comments
- Transparent false positive management

**Slack failure notifications** (from MCO)

- `reporter_config` with channel alerts for disruptive failures
- Useful once MCN has a team Slack channel

## 5. Linting Comparison Across Repos

### golangci-lint Linter Counts

- **Submariner operator**: 60+ linters (most comprehensive)
- **ovn-kubernetes**: 11 linters (focused, practical)
- **CNO**: 12 linters (minimal)
- **CNCC**: 7 linters (minimal)
- **MCO**: not surveyed in detail
- **controller-runtime**: standard Go linting

### Notable Linter Choices

**ovn-kubernetes** uniquely enforces:

- `importas` — enforces Kubernetes import aliases (corev1, metav1,
  apierrors, etc.)
- `testifylint` — testify-specific linting
- `thelper` — test helper function conventions

**CNO** uses `openshift/build-machinery-go` shared Makefile includes
for standardized verify/update/test/build targets.

**MCN recommendation**: Start with Submariner's comprehensive config
(already planned). Consider adding `importas` from ovn-kubernetes
to enforce consistent K8s import aliases from day one.

## 6. AI/Agent Automation Across Repos

- **ovn-kubernetes**: CodeRabbit AI review (`.coderabbit.yml`)
- **cluster-ingress-operator**: CodeRabbit + `AGENTS.md` for AI
  context
- **machine-config-operator**: Claude Code (`.claude/` directory)
  with custom commands for test automation and migration
- **All other OpenShift repos**: no AI automation

**MCN recommendation**: Our planned Claude-based AI reviews (security,
RBAC, release notes) are more targeted than CodeRabbit's general
review. Consider also adding an `AGENTS.md` alongside CLAUDE.md
for broader AI agent compatibility.

## 7. Testing Framework Comparison

### Unit Testing

- **ovn-kubernetes**: 269 test files, Ginkgo/Gomega, Coveralls
- **CNO**: standard Go testing via `hack/test-go.sh`
- **MCO**: standard Go testing
- **Submariner**: Ginkgo/Gomega with JUnit + SonarQube

### E2E Testing

- **ovn-kubernetes**: 47 e2e files, Ginkgo + K8s e2e framework,
  KIND clusters, 45+ matrix combinations
- **CNO**: no in-repo e2e — relies entirely on Prow step registry
  workflows
- **Submariner**: shipyard e2e framework, KIND clusters, feature
  matrix

**MCN recommendation**: Build in-repo e2e tests using Ginkgo + KIND
(like ovn-kubernetes and Submariner). Supplement with Prow step
registry workflows for real cloud testing.

## 8. Container Build Patterns

### Image Count per Repo

- **ovn-kubernetes**: 3 images (base, main, microshift)
- **CNO**: 1 image (3 binaries in one image)
- **Submariner operator**: 2 images (operator, bundle)
- **multus-cni**: 2 images (main, microshift)

### Base Images

- **OpenShift repos**: `registry.ci.openshift.org/ocp/builder:rhel-9-golang-*`
  (build), `registry.ci.openshift.org/ocp/4.XX:base-rhel9` (runtime)
- **Upstream**: golang official image (build), Fedora/Ubuntu/distroless
  (runtime)
- **Submariner**: shipyard-dapper-base (build), scratch (runtime)

**MCN recommendation**: Use `ocp/builder` and `ocp/base-rhel9` for
downstream (matching CNO pattern). Use golang + distroless for
upstream. This dual-Dockerfile approach matches the ecosystem.

## 9. Dependency Management Gap

**No OpenShift networking repo uses Dependabot or Renovate.**
Dependency updates are managed manually or via rebasebot for
upstream syncs.

This is a gap compared to Submariner (which uses Dependabot) and
modern upstream projects (which use Dependabot or Renovate).

**MCN recommendation**: Add Dependabot from day one (already
planned). This puts MCN ahead of the OpenShift networking repos
in dependency hygiene.

## 10. Updated MCN CI Scaffold Additions

Based on this survey, add to the CI scaffold plan:

### New Files to Add

- `AGENTS.md` — AI agent context document (from
  cluster-ingress-operator pattern)
- `.github/labeler.yml` — PR auto-labeling by file path (from
  ovn-kubernetes pattern)
- `hack/verify-codegen.sh` — codegen verification script (from
  MCO pattern)
- `.coderabbit.yml` — optional CodeRabbit config if desired
  alongside Claude reviews

### New GHA Workflow Jobs to Add

- `license-headers` — Apache SkyWalking Eyes license check (from
  ovn-kubernetes)
- `pr-labeler` — auto-label PRs by changed files
- `api-diff` — go-apidiff for breaking API change detection (from
  controller-runtime)

### New Prow Jobs to Plan For

- `verify-deps` — go.mod/vendor consistency
- `security` — openshift-ci-security workflow
- `e2e-aws-ovn` — basic AWS e2e
- `e2e-aws-ovn-upgrade` — upgrade testing
- Skip/run-if-changed filters on all e2e jobs

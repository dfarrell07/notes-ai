---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, metallb, contour, prometheus-operator]
---

# Deep Dive: MetalLB, Contour, Prometheus Operator

File-level deep dives. MetalLB (15+ files) and Contour (25+ files)
completed. Prometheus Operator results pending.

## MetalLB — Key Findings

### 1. CIFuzz with SARIF Upload to Code Scanning

File: `.github/workflows/cifuzz.yml`

OSS-Fuzz CIFuzz runs 600s per PR. The key differentiator: results
uploaded as SARIF to GitHub Code Scanning via
`github/codeql-action/upload-sarif`. Fuzz findings become trackable
code-scanning alerts, not ephemeral CI logs.

4 fuzz targets: BGP OPEN message parsing, CIDR parsing, BGP
community string parsing, ConfigMap-to-CRD YAML decoding.

**MCN**: Enroll in OSS-Fuzz. Add SARIF upload — turns fuzz crashes
into tracked security alerts.

### 2. OPA/Rego Policy Tests for Helm Charts (Conftest)

Files: `charts/metallb/policy/*.rego`

Uses Conftest to validate Helm chart rendered output against Rego
policies. Policies enforce:

- Correct serviceAccountName on DaemonSet
- Specific env var at specific index position
- Node selectors include `kubernetes.io/os: linux`
- Master node tolerations present
- PSP rules in ClusterRoles

Run via: `helm conftest charts/metallb/ -p charts/metallb/policy/`

**MCN**: Catches Helm template regressions that standard linting
misses. Worth adopting if MCN ships Helm charts.

### 3. Periodic Govulncheck Every 6 Hours

File: `.github/workflows/vulncheck_periodic.yaml`

`cron: '0 */6 * * *'` — most aggressive vulnerability scanning
cadence found. Independent from the PR-time govulncheck.

**MCN**: Run govulncheck periodically (daily or 6-hourly) to catch
newly disclosed CVEs between commits.

### 4. PR Body Release-Note Block Enforcement

File: `.github/workflows/classify.yaml`

PRs must contain a ` ```release-note ``` ` fenced code block in the
body. CI fails if this section is deleted from the PR template.

**MCN**: Simple enforcement ensuring release note content is always
present.

### 5. Default-Deny NetworkPolicies in Development

Files: `config/networkpolicies/*.yaml`

Ships and applies default-deny NetworkPolicies during development.
Controller policy only allows egress to port 6443 (API server) and
ingress on metrics/webhook ports.

**MCN**: Test with restrictive NetworkPolicies from day one.

### 6. BuildKit Bind Mounts in Dockerfiles

Files: `controller/Dockerfile`, `speaker/Dockerfile`

Uses `--mount=type=bind` instead of COPY for source, and
`--mount=type=cache` for Go module/build caches. Avoids copying
entire source tree into build context.

**MCN**: More efficient Docker builds. Adopt for upstream
Dockerfiles.

## Contour — Key Findings

### 7. Per-PR Changelog Files — Best Pattern Found

Files: `hack/actions/check-changefile-exists.go`,
`hack/release/prepare-release.go`, `design/changelog.md`

Every PR must include a file:
`changelogs/unreleased/{PR#}-{author}-{category}.md`

Categories: major, minor, small, docs, infra, deprecation.

CI validates:

- File exists with correct naming convention
- File is not empty
- Category in filename matches PR's `release-note/*` label

At release time, `prepare-release.go` reads all unreleased files,
categorizes them, sorts contributors (filtering known maintainers),
and renders via Go template into `CHANGELOG-vX.Y.Z.md`.

For prereleases, files are preserved (not deleted) so they can be
re-aggregated for GA.

**MCN**: Best changelog system found across all 52+ projects.
Eliminates single-person release note bottleneck. CI enforcement
prevents changelog debt. Category-based separation provides
structured release notes without manual sorting.

### 8. Multi-Branch Trivy Scanning via Matrix

File: `.github/workflows/trivy-scan.yaml`

Weekly Trivy scans across 4 branches simultaneously:

```yaml
strategy:
  matrix:
    branch: [main, release-1.33, release-1.32, release-1.31]
```

Each job uploads SARIF to GitHub Code Scanning. Catches CVEs in
all supported release branches.

**MCN**: Trivial workflow change to scan all supported branches.

### 9. Gateway API Conformance Reports as Release Artifacts

Files: `.github/workflows/build_tag.yaml`,
`test/conformance/gatewayapi/gateway_conformance_test.go`

On tag push, automatically runs Gateway API conformance tests and
generates structured YAML conformance report. Dual-mode runner:
uses in-tree tests if Gateway API version matches, otherwise
clones upstream for latest spec testing.

Tests 3 profiles: HTTP, TLS, GRPC.

**MCN**: If MCN implements Gateway API, this is the pattern.

### 10. Automated Dependency Bump Tools

Files: `hack/actions/bump-go-version/main.go`,
`hack/actions/bump-envoy-version/main.go`

Custom Go programs that:

1. Query API for latest patch release
2. Fetch Docker Hub image digest hash
3. Update Makefile `BUILD_BASE_IMAGE` with `@sha256:` digest pin
4. Update version across workflow files
5. Run `go mod tidy`
6. Auto-generate changelog file

**MCN**: The digest-pinned base image pattern
(`golang:1.26.2@sha256:...`) with automated updates is more secure
than version tags alone.

### 11. Compatibility Matrix as YAML

File: `versions.yaml`

Machine-readable YAML tracking every release version with exact
dependencies (Envoy, K8s versions, Gateway API, operator, support
status).

```yaml
- version: v1.33.4
  supported: "true"
  dependencies:
    envoy: "1.35.10"
    kubernetes: ["1.34", "1.33", "1.32"]
    gateway-api: ["1.3.0"]
```

**MCN**: Move compatibility information into structured YAML that
feeds documentation, CI matrix, and compatibility checks.

### 12. E2E Benchmark with Scatter Plot Generation

File: `test/e2e/bench/bench_test.go`

Measures "time to service available" as 1000 services are created.
Uses Ginkgo's `gmeasure` experiment framework. Generates both CSV
data and PNG scatter plots using `gonum.org/v1/plot`.

**MCN**: Novel — embedded visualization generation in test suite.

### 13. Upgrade Tests with Success Rate Thresholds

File: `test/e2e/upgrade/upgrade_test.go`

Deploys previous release, creates routes, upgrades while
continuously polling. Asserts minimum success rate:

- 90% for standalone Contour
- 80% for gateway provisioner (documented known issue)

**MCN**: Replace binary pass/fail upgrade testing with percentage-
based success rate assertions.

### 14. Memory Usage Regression Testing

File: `test/e2e/incluster/memory_usage_test.go`

Creates 100 HTTPProxy resources with 5 header match conditions
each. Asserts no container restarts within 10-second window.
Regression test for a specific memory explosion bug.

**MCN**: Pattern for catching resource consumption regressions.

### 15. golangci-lint — Regex Import Aliases

File: `.golangci.yml`

Regex-based import alias enforcement for Envoy packages:

```yaml
importas:
  alias:
    - pkg: github.com/envoyproxy/go-control-plane/...
      alias: envoy_config_${1}_${2}
  no-unaliased: true
```

Also bans `http.DefaultTransport` via `forbidigo`.

**MCN**: The regex import alias pattern is useful for any project
with deeply nested dependency packages.

## Top MCN Takeaways (Combined)

1. **Per-PR changelog files** (Contour) — best release note system
2. **CIFuzz + SARIF upload** (MetalLB) — fuzz findings as tracked
   security alerts
3. **Multi-branch Trivy scanning** (Contour) — scan all supported
   branches
4. **OPA/Rego Helm chart policies** (MetalLB) — catch template
   regressions
5. **Compatibility matrix YAML** (Contour) — machine-readable
   version tracking
6. **Percentage-based upgrade thresholds** (Contour) — realistic
   upgrade testing
7. **Periodic govulncheck** (MetalLB) — catch CVEs between commits
8. **Digest-pinned base images** (Contour) — supply chain security

## Prometheus Operator — Key Findings

### 16. Feature Gates as Prometheus Metrics

File: `pkg/operator/feature_gates.go`

Feature gate map implements `prometheus.Collector`, exporting each
gate as a gauge metric. Query
`prometheus_operator_feature_gate{name="..."}` in production to see
which gates are active.

**MCN**: If MCN has feature gates, export them as Prometheus
metrics for SRE visibility.

### 17. Centralized Toolchain via `.github/env`

File: `.github/env`

```text
golang-version=1.26
kind-version=v0.31.0
kind-image=kindest/node:v1.35.1
```

Every workflow imports via `cat ".github/env" >> "$GITHUB_ENV"`.
Single file change updates Go/Kind/image versions across all CI.

**MCN**: Simpler than `.go-version` + per-workflow pinning. One
file controls all tool versions.

### 18. Golden File Testing + Domain Validator

Files: `scripts/update-golden-files.sh`, 499 golden files in
`pkg/prometheus/testdata/*.golden`

Two layers: (1) unit tests compare generated config against golden
files, (2) `promtool check config --syntax-only` validates every
golden file is valid Prometheus config.

**MCN**: If MCN generates complex configs, use golden file testing
with a domain-specific validator. Catches both regressions AND
invalid test fixtures.

### 19. Stripped-Down CRDs for kubectl Apply

File: Makefile

jq removes all `description` fields from CRDs to stay under
kubectl's 262144 byte annotation limit. Published as alternative
release asset alongside full CRDs.

**MCN**: If MCN CRDs have extensive docs, provide stripped
alternatives for large CRD workaround.

### 20. promlinter for Metrics Naming

File: Makefile (`check-metrics` target)

`promlinter lint .` validates Prometheus metrics follow naming
conventions (unit suffixes, `_total` for counters, snake_case).

**MCN**: If MCN exports Prometheus metrics, add promlinter. None
of the other deep-dived projects lint metric naming.

### 21. Upgrade Path Testing Against Previous Minor

File: `test/e2e/upgradepath_test.go`

Creates two Framework instances — previous stable version
(auto-discovered from VERSION file on previous release branch)
and current version. Deploys at N-1, creates workloads, upgrades
to N, verifies health.

**MCN**: Automated N-1 to N upgrade testing catches backwards
compatibility breaks.

### 22. Dependabot K8s Grouping + Auto-Merge

Files: `.github/dependabot.yml`,
`.github/workflows/automerge-dependabot.yaml`

Groups all `k8s.io/*` deps into single PR. Auto-approves and
merges patch/minor updates. Major versions require manual review.

**MCN**: Reduces Dependabot PR noise. Auto-merge for safe updates
frees maintainer time.

### 23. golangci-lint — Novel Linters

File: `.golangci.yml`

Notable linters not common elsewhere:

- `exptostd` — use stdlib over experimental packages
- `iotamixing` — catch mixed iota const blocks
- `modernize` — use modern Go constructs
- `nilnesserr` — nil check before error return
- `recvcheck` — consistent receiver names
- depguard bans `sort` package, requires `slices` instead

### 24. Kind Cluster Tuning

File: `test/e2e/kind-conf.yaml`

Reduces `kubelet.syncFrequency` from 60s to 10s. Workers assigned
to fictitious availability zones for topology testing.

**MCN**: Faster E2E tests. The zone labeling pattern is useful
for testing topology-aware features.

### 25. Secret Obfuscation in E2E Diagnostics

File: `test/framework/context.go`

On test failure, collects all resources and logs. Secrets have
data replaced with `obfuscated` before saving.

**MCN**: Collect comprehensive diagnostics on E2E failure but
obfuscate secret values.

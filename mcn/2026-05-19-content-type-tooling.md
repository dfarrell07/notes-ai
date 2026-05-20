---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [linting, testing, tooling, ci, security, content-types]
---

# MCN Content Types and Best Tooling for Each

Every content type that will exist in the MCN operator repo, with the
best modern tool for linting, testing, or validating it. Tools are
tiered: must-have, recommended, nice-to-have.

## Content Type Inventory

An MCN operator repo will contain:

- Go source (controllers, API types, webhooks, utilities)
- Go tests (unit, integration, e2e, fuzz)
- Generated Go code (deepcopy, clients)
- CRD YAML definitions
- Kubernetes manifest YAML (RBAC, deployments, services)
- Kustomize overlays
- Dockerfiles (upstream + Konflux)
- Shell scripts (hack/, scripts/)
- Makefiles
- Markdown (README, docs, contributing guides)
- GitHub Actions workflow YAML
- Tekton pipeline YAML (for Konflux)
- OLM bundle manifests (CSV, annotations)
- Go module files (go.mod, go.sum)
- License headers (across all source files)
- Git commits (message format)

## 1. Go Source Code

### golangci-lint v2 (v2.12.2)

The standard. Use v2 config format (`version: "2"`). Copy from
Submariner's 248-line config and add these newer linters:

**New linters to enable (not in Submariner's config):**

- `modernize` — suggests newer Go stdlib functions and language
  features (strings.CutPrefix, slices.Contains, `any` instead of
  `interface{}`). All fixes are behavior-preserving and
  auto-applicable with `--fix`.
- `funcorder` — enforces constructors after struct declarations,
  exported methods before unexported. Keeps controller files
  organized.
- `iface` — detects incorrect interface use: `opaque` (returning
  interface but always the same concrete type), `unused`
  (interfaces unused in their package). Good for reducing interface
  pollution in operators.
- `recvcheck` — ensures consistent pointer vs value receivers
  across methods on a type. Controller reconcilers should always
  use pointer receivers.
- `importas` — enforces Kubernetes import aliases (corev1, metav1,
  apierrors). Used by ovn-kubernetes. Prevents alias drift.

**Formatters section** (v2 separates these from linters):
`gci`, `gofmt`, `gofumpt`, `goimports`

### kube-api-linter (KAL) — New Discovery

**Tool**: `kubernetes-sigs/kube-api-linter`
**What**: Linter specifically for Kubernetes API types, enforcing
K8s API Conventions. Ships as golangci-lint v2 module.

Key checks for MCN's CRD types:

- `jsontags` — all fields must have json tags, camelCase validated
- `optionalfields` — optional fields must be pointers with
  omitempty
- `conditions` — correct json tags and markers on Conditions
- `nobools` — booleans should be string enums for evolvability
- `nophase` — discourages Phase fields (K8s anti-pattern)
- `defaults` — fields with default markers must be optional
- `commentstart` — comments should start with serialized field name
- `integers` — only int32/int64 allowed

**Integration**: Requires building custom golangci-lint binary via
`.custom-gcl.yml`. Then integrates with standard config. Supports
`--fix` for auto-fixes.

**Priority**: Must-have. This catches what human API reviewers flag.
OpenShift API repo already uses it.

### gosec (via golangci-lint)

Run through golangci-lint (10x faster than standalone). Current
version v2.22.11 includes new rules G116, G124, G708-G710. Only
run standalone if you need dedicated SARIF output for security
dashboards.

## 2. Go Tests

### Unit/Integration Testing

- **envtest** (controller-runtime) — for controller reconciler
  tests. Runs local API server + etcd, starts in seconds, no
  Docker needed. Tests reconcile logic against real API server.
  Standard for kubebuilder-scaffolded operators.
- **Ginkgo/Gomega** — BDD-style test framework. Kubebuilder
  scaffolding generates Ginkgo tests. K8s e2e framework uses
  Ginkgo. Stay with this.
- **Standard Go testing** — for utility/helper functions that don't
  need Ginkgo's structure.

### E2E Testing

- **KIND clusters** — backbone for GHA-based e2e
- **Ginkgo + K8s e2e framework** — standard e2e approach
- **Prow step registry** — for real cloud e2e (AWS, GCP, Azure)

### Fuzz Testing — New for MCN

Go 1.18+ native fuzzing (`testing.F`) for webhook validation:

```go
func FuzzValidateCreate(f *testing.F) {
    f.Add(validAdmissionReviewBytes)
    f.Fuzz(func(t *testing.T, data []byte) {
        // Call webhook handler, check for panics
    })
}
```

Catches panics on malformed input, nil pointer dereferences,
validation edge cases. Run with
`go test -fuzz FuzzValidateCreate -fuzztime 60s`.

**Priority**: Recommended once webhooks exist.

### Coverage

- **Codecov** — most popular in K8s ecosystem. Rich PR comments
  with line-level coverage. Free for open source.
- **Coveralls** — simpler, used by ovn-kubernetes.
- **SonarQube** — used by Submariner. Self-hosted, combines SAST +
  coverage + quality gates.

**Recommendation**: Codecov for simplicity and ecosystem alignment.

## 3. Generated Go Code

- golangci-lint v2 default: `exclusions.generated: strict` skips
  generated files entirely
- CI check: regenerate with controller-gen, diff against committed
  code. MCO's "copy-regenerate-diff" pattern is most robust.
- Don't lint generated code — waste of CI time and produces
  irrelevant noise.

## 4. CRD YAML Definitions

### kubeconform — Replaces kubeval

kubeval is abandoned (last schema update 2020, K8s <= 1.18.1).
kubeconform is the successor: 5x faster, supports CRDs, up-to-date
schemas.

```bash
kubeconform -summary -strict -kubernetes-version 1.31.0 \
  config/crd/bases/
```

**Priority**: Must-have.

### CRD Schema Backward Compatibility

No single tool does this. Recommended pipeline:

1. Extract CRD schemas to OpenAPI JSON Schema
2. Diff with **oasdiff** (450+ breaking change categories)
3. Validate existing CRs against new schema with kubeconform

**Priority**: Recommended once CRD API stabilizes beyond v1alpha1.

## 5. Kubernetes Manifest YAML (RBAC, Deployments, etc.)

### Three-layer validation

1. **yamllint** — generic YAML syntax and style (baseline)
2. **kubeconform** — validates against K8s API schemas
3. **kube-linter** (StackRox/Red Hat) — 40+ security checks:
   running as root, missing resource limits, missing probes,
   `latest` tag, writable root filesystem, etc.

```bash
kube-linter lint config/ deploy/
```

**Priority**: All three are must-have for an operator repo.

### Kustomize Overlays

No dedicated linter needed. Validate rendered output:

```bash
kustomize build config/default | kubeconform -summary
```

## 6. Dockerfiles

### hadolint (v2.14.0)

Still unchallenged as the Dockerfile linter. Embeds ShellCheck for
inline bash in RUN instructions. Catches: missing version pinning,
COPY --chown, unnecessary sudo, apt-get without clean, etc.

```bash
hadolint Dockerfile package/Dockerfile.*
```

**Priority**: Must-have.

## 7. Shell Scripts

### ShellCheck (v0.11.0)

No real alternative exists. Catches: unquoted variables, useless
use of cat, unreachable code, numeric comparison issues.

```bash
find . -name "*.sh" -not -path "./vendor/*" -exec shellcheck {} +
```

### shfmt — Formatter

Companion to ShellCheck. Enforces consistent formatting.

```bash
shfmt -d -i 2 scripts/
```

**Priority**: ShellCheck is must-have. shfmt is recommended.

### bats-core (v1.13.0)

Shell script testing framework. Worth adding if MCN has complex
setup/deployment scripts.

**Priority**: Nice-to-have.

## 8. Makefiles

### checkmake

Only Makefile linter available. Catches missing `.PHONY` targets,
missing descriptions. Actively maintained (March 2026).

**Priority**: Nice-to-have. Low effort but low value.

## 9. Markdown

### markdownlint-cli2 (v0.22.1)

Recommended over markdownlint-cli. Same author as the library,
config-first design, shared config with VS Code extension.

### lychee — Link Checker

Recommended replacement for the now-deprecated
markdown-link-check. Written in Rust, very fast, native GHA action.
Replaces `gaurav-nelson/github-action-markdown-link-check`.

```bash
lychee --cache --max-concurrency 10 "**/*.md"
```

**Priority**: markdownlint-cli2 is must-have. lychee is recommended
(replaces our planned markdown-link-check).

## 10. GitHub Actions Workflow YAML

### actionlint (v1.7.12)

Structural correctness, expression type checking, action
input/output validation. Auto-pipes `run:` scripts through
ShellCheck.

```bash
actionlint .github/workflows/*.yml
```

### zizmor — GHA Security Scanner (New)

24 audit rules for GitHub Actions security. Catches template
injection, unpinned action versions, secrets exposure, dangerous
triggers. Grafana Labs deployed it across 2000+ repos. Academic
paper (March 2026) found widest coverage of any GHA scanner.
Outputs SARIF for GitHub Advanced Security integration.

```bash
zizmor --format sarif .github/workflows/ > zizmor.sarif
```

**Priority**: Both are must-have. actionlint for correctness,
zizmor for security.

## 11. Tekton Pipeline YAML

### IBM/tekton-lint

Only dedicated Tekton linter. Validates parameter names/types,
task references, resource references. Requires Node.js.

```bash
npx @ibm/tekton-lint@latest '.tekton/**/*.yaml'
```

**Priority**: Nice-to-have (only relevant once Konflux pipelines
exist).

## 12. OLM Bundle Manifests

### operator-sdk bundle validate

The standard, no alternative. Use
`--select-optional suite=operatorframework` for full validation.

```bash
operator-sdk bundle validate bundle/ \
  --select-optional suite=operatorframework
```

**Priority**: Must-have once OLM bundle exists.

## 13. Go Module Files

### govulncheck (v1.1.4)

Primary vulnerability scanner. Symbol-level reachability analysis
means far fewer false positives than module-level scanners. SARIF
output via `-format sarif`. Official Go team tool.

```bash
govulncheck ./...
govulncheck -format sarif ./... > govulncheck.sarif
```

### go-licenses (v2.0.0)

Dependency license scanning and compliance. Required for any
project shipping binaries.

```bash
go-licenses check ./...
```

### nancy (Sonatype)

Optional supplement using different vulnerability database.
Supports `.nancy-ignore` for false positive suppression.

**Priority**: govulncheck is must-have. go-licenses is recommended.
nancy is nice-to-have.

## 14. License Headers

### Apache SkyWalking Eyes

Most comprehensive for multi-language repos: check/fix headers +
dependency license auditing. Supports Go, shell, Python, Makefile,
Dockerfile. Native GHA action. Used by ovn-kubernetes.

```bash
license-eye header check
```

Alternative: `goheader` via golangci-lint (Go files only).

**Priority**: Recommended. One of SkyWalking Eyes (multi-language)
or goheader (Go-only via golangci-lint) — not both.

## 15. Git Commits

### conventionalcommit/commitlint (Go)

Pure Go binary for conventional commit validation. No Node.js or
Python dependency. Supports custom rules via Go interface.

Alternative: `gitlint` (Python, used by Submariner).

**Priority**: One of these is must-have. commitlint if you want
conventional commits for auto-changelog; gitlint if you want
Submariner compatibility.

## 16. API Compatibility

### go-apidiff

Compares exported Go API surface between two git commits. Reports
incompatible changes (removed/changed exported types, functions,
methods). GHA action available.

```bash
go-apidiff $(git merge-base HEAD main)
```

**Priority**: Must-have for CRD API types.

### OSSF Scorecard

Weekly supply chain security analysis. Uploads SARIF to GitHub
Code Scanning. Publishes score to OpenSSF. Used by
controller-runtime.

**Priority**: Recommended. Low effort, high visibility.

## Summary — Tiered Recommendations

### Must-Have (Day One)

| Content | Tool |
| --- | --- |
| Go source | golangci-lint v2 (60+ linters) |
| Go CRD types | kube-api-linter (KAL) |
| Go tests | Ginkgo/Gomega + envtest |
| Go vulnerabilities | govulncheck with SARIF |
| Go security | gosec (via golangci-lint) |
| Go API compat | go-apidiff |
| K8s manifests | kubeconform + kube-linter |
| YAML syntax | yamllint |
| Dockerfiles | hadolint |
| Shell scripts | ShellCheck |
| Markdown | markdownlint-cli2 |
| GHA workflows | actionlint + zizmor |
| Commits | gitlint or commitlint |
| Coverage | Codecov |

### Recommended (Phase 2)

| Content | Tool |
| --- | --- |
| Go formatting | shfmt for shell, gofumpt via golangci-lint |
| Go licenses | go-licenses |
| License headers | Apache SkyWalking Eyes |
| Markdown links | lychee (replaces markdown-link-check) |
| Supply chain | OSSF Scorecard |
| SAST | CodeQL variant analysis |
| Fuzz testing | Go native fuzzing for webhooks |
| Go modules | nancy (supplement to govulncheck) |

### Nice-to-Have (Phase 3+)

| Content | Tool |
| --- | --- |
| Makefiles | checkmake |
| Tekton YAML | IBM/tekton-lint |
| OLM bundles | operator-sdk bundle validate |
| CRD compat | oasdiff pipeline |
| Shell tests | bats-core |
| Go staleness | go-mod-outdated |
| K8s RBAC | Checkov IaC policy checks |

## Updates to CI Scaffold Plan

Based on this research, the following changes to the CI scaffold
plan (section 12 of automation-planning.md):

### New Tools to Add

1. **kube-api-linter (KAL)** — add to golangci-lint config via
   `.custom-gcl.yml` module integration
2. **kubeconform** — add as linting.yml job for K8s manifest
   validation
3. **kube-linter** — add as linting.yml job for K8s security
   checks
4. **hadolint** — add as linting.yml job for Dockerfile linting
5. **zizmor** — add as linting.yml job for GHA security scanning
6. **go-apidiff** — add as linting.yml job for API compatibility
7. **lychee** — replace markdown-link-check in periodic.yml
8. **actionlint** — add as linting.yml job for workflow validation

### New Linters for .golangci.yml

Add to the enable list: `modernize`, `funcorder`, `iface`,
`recvcheck`, `importas`

### Revised Linting Workflow Job Count

Was 12 jobs, now 18 with new additions:

1. apply-suggestions-commits
2. gitlint
3. golangci-lint (includes KAL, gosec, modernize, etc.)
4. markdownlint
5. shellcheck
6. yamllint
7. govulncheck
8. variant-analysis (CodeQL)
9. vulnerability-scan (Anchore)
10. crds (generated code freshness)
11. licenses (google/go-licenses)
12. kubeconform (K8s manifest validation)
13. kube-linter (K8s security checks)
14. hadolint (Dockerfile linting)
15. actionlint (GHA workflow validation)
16. zizmor (GHA security scanning)
17. go-apidiff (API compatibility)
18. license-headers (SkyWalking Eyes)

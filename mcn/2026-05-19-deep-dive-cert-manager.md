---
date: 2026-05-19T00:00:00Z
topic: mcn
tags: [ci, deep-dive, cert-manager, makefile, klone, renovate, scorecard]
---

# Deep Dive: cert-manager Repo Automation

File-level deep dive of github.com/cert-manager/cert-manager.
29 files read. cert-manager has the most sophisticated modular
build system in the K8s ecosystem.

## Key Findings for MCN

### 1. Modular Makefile System (klone)

Files: `Makefile`, `klone.yaml`, `make/_shared/*/`

Shared Makefile modules imported from
`github.com/cert-manager/makefile-modules`. The `klone.yaml` file
pins 9 modules at specific git hashes:

- boilerplate, generate-verify, go, help, klone, licenses,
  repository-base, tools, helm

Include order enforced: `00_mod.mk` (variables) -> shared
`00_mod.mk` -> shared `01_mod.mk` -> `02_mod.mk` (targets) ->
shared `02_mod.mk`.

**Self-upgrading**: daily cron runs `make upgrade-klone` +
`make generate`, auto-creates PRs with labels `ok-to-test,
skip-review, release-note-none, kind/cleanup`.

Shell hardened: `--norc -uo pipefail -c`,
`--warn-undefined-variables`, `--no-builtin-rules`.

**MCN**: The modular Makefile concept is interesting but heavy
for a single repo. The shell hardening and include ordering
patterns are worth copying. The self-upgrade bot pattern is
excellent if MCN ever shares build infra across repos.

### 2. Vendored Go Toolchain

File: `make/_shared/tools/00_mod.mk`

Downloads Go 1.26.3 into `_bin/` for reproducibility. In CI,
tool downloads go to `_bin/`; locally they cache to
`~/.cache/makefile-modules`.

**MCN**: Consider pinning Go version for reproducibility. The
`_bin/` pattern avoids polluting system Go installations.

### 3. OSSF Scorecard — 8.5/10

File: `.github/workflows/scorecards.yml`

Weekly run (Saturdays 13:43 UTC). Uploads SARIF to GitHub code
scanning. Publishes results for badge generation.

Perfect 10s on: Code-Review, Dependency-Update-Tool, Maintained,
Dangerous-Workflow, Token-Permissions, Binary-Artifacts,
Pinned-Dependencies, License, Fuzzing, Vulnerabilities, CI-Tests,
Contributors.

Weak: Signed-Releases (0), SAST (0), CII-Best-Practices (5).

**MCN**: Adopt from day one. Single workflow file, instant
visibility into security posture.

### 4. octo-sts — Short-Lived GitHub Tokens

Files: `.github/chainguard/make-self-upgrade.sts.yaml`,
`.github/workflows/make-self-upgrade.yaml`

Uses Chainguard's octo-sts for short-lived GitHub tokens instead
of long-lived PATs. OIDC-based
(`issuer: https://token.actions.githubusercontent.com`).

Subject pattern restricts to main/master branches. Permissions:
`contents: write`, `pull_requests: write`, `workflows: write`.

Workflow exchanges `id-token: write` for a scoped token via
`octo-sts/action@v1.1.1`.

**MCN**: Consider for any cross-repo automation. Better security
than PATs stored as secrets.

### 5. golangci-lint — 48 Linters

File: `.golangci.yaml`

Notable unique choices:

- `gosmopolitan` — detects non-i18n-friendly code
- `exhaustive` with `default-signifies-exhaustive: true` —
  requires exhaustive switch coverage
- `bidichk` — detects Unicode bidirectional control characters
  (supply chain security)
- `sloglint` — structured logging conventions
- `promlinter` — Prometheus metrics naming validation
- `protogetter` — protobuf getter enforcement

gosec has 3 specific exclusions: G101 (hardcoded creds), G204
(command exec), G306 (file permissions) — all documented.

**MCN**: The `bidichk` linter is a nice supply chain defense
(invisible Unicode attacks). `exhaustive` is good for ensuring
complete switch/case coverage. `promlinter` is useful if MCN
exports Prometheus metrics.

### 6. E2E Infrastructure — 10 Components

File: `make/e2e-setup.mk`

Deploys 10 infrastructure components for e2e testing:

1. cert-manager (from local builds)
2. Pebble (ACME server, forked with Ed25519 support)
3. BIND9 (DNS server for DNS-01 challenges)
4. nginx-ingress (HTTP-01 Ingress tests)
5. kgateway (HTTP-01 GatewayAPI tests)
6. Vault (HashiCorp, for Vault issuer tests)
7. Kyverno (policy enforcement testing)
8. sample-external-issuer (conformance)
9. sample-webhook (webhook testing)
10. Gateway API CRDs (v1.5.1 experimental)

40 Ginkgo nodes in parallel. OpenShift support via
`E2E_OPENSHIFT=true`. K8s images pinned by SHA256 digest.

**MCN**: The comprehensive e2e infrastructure is a good reference.
MCN will need similar test infrastructure for BGP peers, EVPN
fabric, etc.

### 7. Pin-by-Digest Everything

Files: `make/base_images.mk`, e2e setup files

ALL base images, test infrastructure images, and GitHub Actions
are pinned by SHA256 digest. Not just version tags — actual
digests. E2e setup uses `crane` to verify remote image tag+digest
still match what is pinned (detects tag mutation attacks).

Auto-updated by `hack/latest-base-images.sh` + Renovate custom
manager.

**MCN**: Best practice. SHA-pin all container image references
and GHA action references.

### 8. Renovate Config — Branch-Aware Updates

File: `.github/renovate.json5`

Extends shared config from makefile-modules repo. Key patterns:

- Release branches allow only patch/pin/digest updates
- Custom regex managers for Docker image digests in Makefiles
- Post-upgrade task: when Kind is updated, automatically
  regenerates Kind node image references
- Base image updates grouped into single "Base Images" PR

**MCN**: The release-branch update restriction (only safe
updates) is important once MCN has release branches.

### 9. Code Generation — 8 Generators

File: `hack/k8s-codegen.sh`

Runs 8 K8s code generators:

1. client-gen — typed clients
2. deepcopy-gen — DeepCopy methods
3. informer-gen — shared informers
4. lister-gen — typed listers
5. defaulter-gen — webhook defaults
6. conversion-gen — internal/external conversion
7. openapi-gen — OpenAPI specs
8. applyconfiguration-gen — SSA apply configs

CRDs generated from Helm chart using `helm template` + custom
`hack/extractcrd/main.go`.

**MCN**: Most of these are only needed for complex operators.
Start with controller-gen (CRDs + DeepCopy). Add client-gen and
informer-gen if MCN needs typed client libraries.

### 10. Container Build — Hard Links for Context Size

File: `make/containers.mk`

Separate `Containerfile.*` per binary. Uses hard-linked binaries
to avoid 1.1GB Docker context copies. Per-arch build contexts.

`CGO_ENABLED=0`, `-trimpath`, `-w -s` (strip debug), version
ldflags.

**MCN**: The hard-link optimization is clever for large builds.
CGO_ENABLED=0 and -trimpath should be standard.

### 11. Release — KMS Signing + Helm Provenance

Files: `make/release.mk`, `gcb/build_cert_manager.yaml`

Google Cloud Build on tag push. KMS key for Helm chart provenance
(`.tgz.prov` files). 5 architectures (amd64, arm64, s390x,
ppc64le, arm). Cosign signing prototyped but commented out.

**MCN**: The KMS-based signing is relevant for downstream
(Konflux handles signing). The 5-arch support shows the full
range of platforms cert-manager supports.

## Top MCN Takeaways

1. **OSSF Scorecard from day one** — single workflow, instant
   security visibility
2. **Pin everything by SHA digest** — container images and GHA
   actions
3. **bidichk linter** — supply chain defense against invisible
   Unicode
4. **exhaustive switch coverage** — catches missing cases
5. **promlinter** — validates Prometheus metric naming conventions
6. **Release branch update restrictions** — only safe updates on
   stable branches
7. **octo-sts for cross-repo automation** — better than PATs

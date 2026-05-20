---
date: 2026-05-20T00:00:00Z
topic: mcn
tags: [cncf, graduated, ci, automation, research]
---

# CNCF Graduated Projects — CI/Automation Survey

36 graduated projects as of May 2026. Goal: survey their
CI/automation practices for MCN adoption. Projects we already
deep-dived are marked; the rest need investigation.

## Projects

| # | Project | Repo | Already Surveyed? |
| --- | --- | --- | --- |
| 1 | Argo | argoproj/argo-workflows | Yes (deep dive) |
| 2 | cert-manager | cert-manager/cert-manager | Yes (deep dive) |
| 3 | Cilium | cilium/cilium | Yes (deep dive) |
| 4 | CloudEvents | cloudevents/spec | No |
| 5 | containerd | containerd/containerd | No |
| 6 | CoreDNS | coredns/coredns | Yes (deep dive) |
| 7 | CRI-O | cri-o/cri-o | No |
| 8 | Crossplane | crossplane/crossplane | Yes (deep dive) |
| 9 | CubeFS | cubefs/cubefs | No |
| 10 | Dapr | dapr/dapr | Yes (deep dive) |
| 11 | Dragonfly | dragonflyoss/dragonfly | No |
| 12 | Envoy | envoyproxy/envoy | No |
| 13 | etcd | etcd-io/etcd | Yes (deep dive) |
| 14 | Falco | falcosecurity/falco | Yes (deep dive) |
| 15 | Fluentd | fluent/fluentd | No |
| 16 | Flux | fluxcd/flux2 | Yes (deep dive) |
| 17 | Harbor | goharbor/harbor | Yes (deep dive) |
| 18 | Helm | helm/helm | Yes (deep dive) |
| 19 | in-toto | in-toto/in-toto | No |
| 20 | Istio | istio/istio | Yes (deep dive) |
| 21 | Jaeger | jaegertracing/jaeger | Yes (deep dive) |
| 22 | KEDA | kedacore/keda | Yes (deep dive) |
| 23 | Knative | knative/serving | Yes (deep dive) |
| 24 | KubeEdge | kubeedge/kubeedge | Yes (deep dive) |
| 25 | Kubernetes | kubernetes/kubernetes | No |
| 26 | Kyverno | kyverno/kyverno | Yes (deep dive) |
| 27 | Linkerd | linkerd/linkerd2 | Yes (deep dive) |
| 28 | OPA | open-policy-agent/opa | Yes (deep dive) |
| 29 | OpenTelemetry | open-telemetry/opentelemetry-collector | No |
| 30 | Prometheus | prometheus/prometheus | Yes (broad survey) |
| 31 | Rook | rook/rook | Yes (deep dive) |
| 32 | SPIFFE | spiffe/spiffe | No |
| 33 | SPIRE | spiffe/spire | No |
| 34 | TUF | theupdateframework/python-tuf | No |
| 35 | TiKV | tikv/tikv | No |
| 36 | Vitess | vitessio/vitess | No |

## Not Yet Surveyed (14 projects)

These still need their CI/automation explored:

1. **CloudEvents** — spec repo, likely minimal CI
2. **containerd** — major container runtime, Go
3. **CRI-O** — K8s container runtime, Go, OpenShift-adjacent
4. **CubeFS** — distributed storage, Go
5. **Dragonfly** — P2P distribution, Go
6. **Envoy** — proxy, C++/Bazel (different build ecosystem)
7. **Fluentd** — logging, Ruby
8. **in-toto** — supply chain security, Python
9. **Kubernetes** — the big one, Go/Prow
10. **OpenTelemetry** — multi-repo org, multi-language
11. **SPIFFE** — spec + reference impl, Go
12. **SPIRE** — SPIFFE runtime, Go
13. **TUF** — update framework, Python
14. **TiKV** — distributed KV store, Rust
15. **Vitess** — MySQL clustering, Go

## Priority for MCN

Most relevant to survey (Go, K8s operator/controller pattern):

1. **containerd** — large Go project, likely mature CI
2. **CRI-O** — Go, OpenShift ecosystem, probably Prow + GHA
3. **Kubernetes** — the reference for Prow, bot automation
4. **SPIRE** — Go, security-focused, likely strict CI
5. **Vitess** — Go, large project, likely sophisticated CI

Lower priority (different language/ecosystem):

- Envoy (C++/Bazel), Fluentd (Ruby), in-toto (Python),
  TUF (Python), TiKV (Rust)

## Findings

Details will be added per project as they are investigated.

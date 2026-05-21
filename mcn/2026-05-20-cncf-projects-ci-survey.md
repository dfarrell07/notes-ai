---
date: 2026-05-20T00:00:00Z
topic: mcn
tags: [cncf, graduated, incubating, ci, automation, research]
---

# CNCF Projects — CI/Automation Survey

36 graduated + 38 incubating projects as of May 2026. Goal:
survey their CI/automation practices for MCN adoption.
Projects we already deep-dived are marked; the rest need
investigation.

## Graduated Projects (36)

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

## Incubating Projects (38)

| # | Project | Repo | Lang | Already Surveyed? |
| --- | --- | --- | --- | --- |
| 1 | Artifact Hub | artifacthub/hub | Go/TS | No |
| 2 | Backstage | backstage/backstage | TS | No |
| 3 | Buildpacks | buildpacks/pack | Go | Yes (deep dive) |
| 4 | Chaos Mesh | chaos-mesh/chaos-mesh | Go | Yes (deep dive) |
| 5 | Cloud Custodian | cloud-custodian/cloud-custodian | Python | No |
| 6 | CNI | containernetworking/cni | Go | No |
| 7 | Contour | projectcontour/contour | Go | Yes (deep dive) |
| 8 | Cortex | cortexproject/cortex | Go | No |
| 9 | Emissary-Ingress | emissary-ingress/emissary | Python | Yes (deep dive) |
| 10 | Flatcar | flatcar/Flatcar | Shell | No |
| 11 | Fluid | fluid-cloudnative/fluid | Go | No |
| 12 | gRPC | grpc/grpc | C++ | No |
| 13 | Karmada | karmada-io/karmada | Go | Yes (deep dive) |
| 14 | Keycloak | keycloak/keycloak | Java | No |
| 15 | KServe | kserve/kserve | Python/Go | No |
| 16 | Kubeflow | kubeflow/kubeflow | Python/Go | No |
| 17 | Kubescape | kubescape/kubescape | Go | No |
| 18 | KubeVela | kubevela/kubevela | Go | No |
| 19 | KubeVirt | kubevirt/kubevirt | Go | Yes (deep dive) |
| 20 | Lima | lima-vm/lima | Go | No |
| 21 | Litmus | litmuschaos/litmus | Go/TS | No |
| 22 | Longhorn | longhorn/longhorn | Go | Yes (deep dive) |
| 23 | Metal3-io | metal3-io/baremetal-operator | Go | No |
| 24 | Microcks | microcks/microcks | Java | No |
| 25 | NATS | nats-io/nats-server | Go | No |
| 26 | Notary | notaryproject/notation | Go | No |
| 27 | OpenCost | opencost/opencost | Go | No |
| 28 | OpenFeature | open-feature/flagd | Go | No |
| 29 | OpenFGA | openfga/openfga | Go | No |
| 30 | OpenKruise | openkruise/kruise | Go | No |
| 31 | OpenYurt | openyurtio/openyurt | Go | No |
| 32 | Operator Framework | operator-framework/operator-sdk | Go | Yes (OLM deep dive) |
| 33 | Strimzi | strimzi/strimzi-kafka-operator | Java | Yes (deep dive) |
| 34 | Tekton | tektoncd/pipeline | Go | Yes (deep dive) |
| 35 | Thanos | thanos-io/thanos | Go | Yes (deep dive) |
| 36 | Volcano | volcano-sh/volcano | Go | Yes (deep dive) |
| 37 | wasmCloud | wasmCloud/wasmCloud | Rust | No |

## Not Yet Surveyed — Graduated (14)

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

## Not Yet Surveyed — Incubating (19)

1. **Artifact Hub** — Go/TS, package discovery
2. **Backstage** — TS, developer portal
3. **Cloud Custodian** — Python, cloud policy
4. **CNI** — Go, networking spec/plugins
5. **Cortex** — Go, Prometheus long-term storage
6. **Flatcar** — Linux distro, Shell/Python
7. **Fluid** — Go, data acceleration for K8s
8. **gRPC** — C++, multi-language RPC
9. **Keycloak** — Java, identity/auth
10. **KServe** — Python/Go, ML inference
11. **Kubeflow** — Python/Go, ML platform
12. **Kubescape** — Go, K8s security
13. **KubeVela** — Go, app delivery
14. **Lima** — Go, Linux VMs for Mac
15. **Litmus** — Go/TS, chaos engineering
16. **Metal3-io** — Go, bare metal K8s provisioning
17. **Microcks** — Java, API mocking
18. **NATS** — Go, messaging system
19. **Notary** — Go, artifact signing
20. **OpenCost** — Go, K8s cost monitoring
21. **OpenFeature** — Go, feature flagging
22. **OpenFGA** — Go, fine-grained auth
23. **OpenKruise** — Go, workload automation
24. **OpenYurt** — Go, edge K8s
25. **wasmCloud** — Rust, WebAssembly platform

## Priority for MCN

Most relevant unsurveyed (Go, K8s operator/controller pattern):

**Graduated:**

1. **CRI-O** — Go, OpenShift ecosystem, probably Prow + GHA
2. **containerd** — large Go project, likely mature CI
3. **Kubernetes** — the reference for Prow, bot automation
4. **SPIRE** — Go, security-focused, likely strict CI
5. **Vitess** — Go, large project, likely sophisticated CI

**Incubating:**

1. **Metal3-io** — Go, K8s operator, OpenShift ecosystem
2. **Kubescape** — Go, K8s security scanning
3. **NATS** — Go, large project, likely mature CI
4. **CNI** — Go, networking (directly relevant domain)
5. **Notary** — Go, supply chain security
6. **Operator Framework** — already surveyed via OLM

Lower priority (different language/ecosystem):

- Envoy (C++), Fluentd (Ruby), in-toto (Python),
  TUF (Python), TiKV (Rust), gRPC (C++), Keycloak (Java),
  Backstage (TS), wasmCloud (Rust)

## Findings

Details will be added per project as they are investigated.

# 13. Sources and verification notes

This guide was assembled from official documentation and project sources. For transparency it separates what was **actually read while writing** from what is **recommended further reading**. Technical facts that move fast (versions, CRD API versions, licensing, product editions) should always be re-checked against the current upstream page.

## Read while writing

| Topic | Source |
|---|---|
| GitLab CI/CD YAML keywords (global keywords, `include`, `rules`, `needs`, `artifacts`, `cache`, `workflow`) | <https://docs.gitlab.com/ci/yaml/> (the first part of this very long page was read; later sections were not) |
| GitLab approval rules, eligible approvers, CODEOWNERS | <https://docs.gitlab.com/user/project/merge_requests/approvals/rules/> |
| GitLab Duo in merge requests (Code Review Flow, Duo Code Review, tiers and caveats) | <https://docs.gitlab.com/user/project/merge_requests/duo_in_merge_requests/> |
| GitLab CI ID tokens and OIDC with Vault | <https://docs.gitlab.com/ci/secrets/id_token_authentication/> |
| Argo CD core concepts | <https://argo-cd.readthedocs.io/en/stable/core_concepts/> |
| Argo CD cluster bootstrapping, app of apps, ApplicationSets | <https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/> |
| Kubernetes Pods | <https://kubernetes.io/docs/concepts/workloads/pods/> (the sidecar and probe sections were truncated in the fetch) |
| Istio security: PeerAuthentication, AuthorizationPolicy, RequestAuthentication | <https://istio.io/latest/docs/concepts/security/> |
| HashiCorp Vault on Kubernetes (Agent Injector, Vault Secrets Operator, CSI provider) | <https://developer.hashicorp.com/vault/docs/platform/k8s> |
| External Secrets Operator concepts (SecretStore, ClusterSecretStore, ExternalSecret) | <https://external-secrets.io/latest/introduction/overview/> |
| Retirement of the ingress-nginx controller (maintenance ended March 2026) | <https://www.kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/> and <https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/> |
| Promtail end of life (2 March 2026) and migration to Grafana Alloy | <https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/> and <https://grafana.com/docs/loki/next/send-data/promtail/> |
| OpenTelemetry Operator auto-instrumentation (`Instrumentation` CRD, `inject-dotnet` annotation) | <https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/> |
| Bitnami catalogue changes of 2025 (images moved to legacy, no further free updates) | Summarised from community and vendor notices; the primary sources are Bitnami's own GitHub issues: confirm there |
| Artifactory OSS versus Docker support, Harbor as a CNCF registry | Summarised from search results (JFrog documentation, third-party comparisons); confirm against JFrog's current edition comparison |

## Not fetched: recommended primary reading

These are the authoritative places to continue. Statements about them in this guide come from general knowledge and should be verified.

| Topic | Where to look |
|---|---|
| GitLab reference architectures, Linux package install, backup/restore | docs.gitlab.com (Install, Administration, Backup and restore) |
| GitLab Runner Kubernetes executor and Helm chart | docs.gitlab.com/runner |
| CloudNativePG (Cluster, Pooler, backup and recovery) | cloudnative-pg.io/documentation |
| Strimzi (Kafka, KafkaNodePool, KRaft, KafkaUser/Topic) | strimzi.io/docs |
| Valkey / Redis operators and licensing | valkey.io, redis.io, and the operator project of your choice |
| Elastic Cloud on Kubernetes (ECK) | elastic.co/guide/en/cloud-on-k8s |
| Grafana Loki, Alloy, Tempo, Grafana provisioning | grafana.com/docs |
| kube-prometheus-stack and Prometheus Operator | prometheus-community.github.io/helm-charts, prometheus-operator.dev |
| OpenTelemetry Collector and .NET SDK | opentelemetry.io/docs |
| Vault Kubernetes auth, JWT auth, database and PKI engines | developer.hashicorp.com/vault/docs |
| OpenBao (open-source Vault fork) | openbao.org |
| cert-manager, MetalLB, Cilium, Rook-Ceph, Longhorn | cert-manager.io, metallb.io, docs.cilium.io, rook.io, longhorn.io |
| Harbor | goharbor.io/docs |
| Gateway API | gateway-api.sigs.k8s.io |
| .NET containers, health checks | learn.microsoft.com (Containerize a .NET app, Health checks in ASP.NET Core) |
| Kubernetes probes, resource management, Pod Security Standards | kubernetes.io/docs |

## What was not done

- None of the manifests, pipelines or commands were executed against a real cluster, GitLab instance, Vault or registry.
- The Mermaid diagrams were written for GitHub's renderer but were not previewed there before delivery; if one fails to render, check the label quoting and report the file.
- No academic sources (university course material) were used; the vendor and project documentation above is more authoritative for these specific tools.

## Versions to confirm before use

Kubernetes, Istio, Argo CD, GitLab, Vault/OpenBao, CloudNativePG, Strimzi (and the Kafka versions it supports), ECK, Loki, Alloy, Tempo, the OpenTelemetry Operator and Collector, and the External Secrets Operator all change quickly. API versions shown here (`v1`, `v1beta2`, `v1alpha1`, `v1beta1`) reflect common usage and may differ in the release you install.

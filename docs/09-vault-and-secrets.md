# 9. HashiCorp Vault and secrets management

## 9.1 The problem

Static secrets in `.env` files, Git history, images and CI variables leak and never rotate. Vault centralises secrets, authenticates **workloads by identity**, applies least-privilege path-based **policies**, writes an **audit log** of every access, and can mint **short-lived dynamic credentials** (for example PostgreSQL users that expire).

> **Licensing note.** Vault's source licence changed to the Business Source License in 2023. A community fork, **OpenBao** (Linux Foundation), keeps an open-source licence and is largely API-compatible. Evaluate both against your licensing policy; the concepts below apply to either.

## 9.2 Architecture

```mermaid
flowchart TB
    subgraph TOOLS["Tools zone"]
        V1["Vault node 1"]
        V2["Vault node 2"]
        V3["Vault node 3"]
        V1 --- V2 --- V3
        V3 --- V1
    end
    AUD["Audit log -> Loki / Elasticsearch"]
    V1 -.-> AUD
    UNSEAL["Transit auto-unseal<br/>(small separate Vault) or HSM"] -.-> TOOLS

    subgraph K8S["Kubernetes"]
        ESO["External Secrets Operator"]
        APP["App pod (ServiceAccount)"]
        SEC["Kubernetes Secret"]
        ESO -->|"creates"| SEC
        SEC -->|"env / volume"| APP
    end
    ESO -->|"Kubernetes auth"| V1
    CI["GitLab CI job"] -->|"JWT auth (id_token)"| V1
    PGSQL[("PostgreSQL")] <-->|"database secrets engine"| V1
    CA["cert-manager"] -->|"PKI engine"| V1
```

Run Vault **outside** the application cluster (three VMs, integrated Raft storage) or in a dedicated tools cluster, so that restoring an application cluster never depends on itself. Running Vault inside the cluster it serves is possible (Helm chart, 3 replicas, Raft) but creates a bootstrap and recovery loop you must plan for.

## 9.3 Installation essentials

| Topic | Guidance |
|---|---|
| **Storage** | Integrated Raft on 3 (or 5) nodes with fast disks; snapshot on a schedule and ship snapshots off the cluster |
| **TLS** | Always; certificates from your internal CA; clients trust that CA |
| **Unseal** | With no cloud KMS on-prem, options are: **Shamir** keys held by multiple people (manual, operationally painful), **transit auto-unseal** from a small separate Vault, or an **HSM** (PKCS#11, a paid feature). Never keep unseal keys in chat or a repo |
| **Audit devices** | Enable at least two (file and syslog/socket) before anything else; ship to your logging stack |
| **Root token** | Use only for bootstrap, then revoke; use SSO/OIDC for humans |
| **Namespaces/paths** | Use a path convention: `kv/<team>/<env>/<service>/...` |

## 9.4 How a pod authenticates (Kubernetes auth method)

```mermaid
sequenceDiagram
    participant P as Pod / controller
    participant V as Vault
    participant K as Kubernetes API
    P->>V: Login with ServiceAccount JWT + role name
    V->>K: TokenReview (is this JWT valid? which SA/namespace?)
    K-->>V: Identity (serviceaccount, namespace)
    V->>V: Match role bindings, attach policies
    V-->>P: Short-lived Vault token
    P->>V: Read secret (policy-checked)
    V-->>P: Secret value
```

(This mechanism is described in HashiCorp's Kubernetes auth method documentation; the Kubernetes platform overview page lists the delivery options in 9.6.)

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc"

vault secrets enable -path=kv -version=2 kv

vault policy write orders-api - <<'EOF'
path "kv/data/orders/prod/orders-api/*" { capabilities = ["read"] }
EOF

vault write auth/kubernetes/role/orders-api \
  bound_service_account_names=orders-api \
  bound_service_account_namespaces=orders-prod \
  policies=orders-api ttl=1h
```

Notes: policies are deny-by-default; KV v2 policy paths contain `/data/` (a frequent "permission denied" cause); a Vault running outside the cluster needs a way to call that cluster's TokenReview API (a reviewer service account token or JWT/OIDC validation against the cluster's issuer).

## 9.5 How CI authenticates (JWT / OIDC)

GitLab ID tokens (chapter 4) let a CI job log in without any stored secret:

```bash
vault auth enable -path=jwt jwt
vault write auth/jwt/config jwks_url="https://gitlab.acme.internal/-/jwks" bound_issuer="https://gitlab.acme.internal"

vault write auth/jwt/role/ci-orders - <<'EOF'
{
  "role_type": "jwt",
  "user_claim": "project_id",
  "bound_audiences": ["https://vault.acme.internal"],
  "bound_claims": { "project_id": "42", "ref_protected": "true" },
  "policies": ["ci-orders"],
  "token_ttl": "10m"
}
EOF
```

Bound claims restrict the role to one project and to **protected refs only**, so a feature branch cannot read production-grade secrets. Prefer stable claims (`project_id`) over mutable names.

## 9.6 Delivering secrets into pods

HashiCorp's Kubernetes platform documentation lists three integrations that avoid application code changes: the **Vault Agent Injector**, the **Vault Secrets Operator (VSO)**, and the **Vault Secrets Store CSI provider**. **External Secrets Operator (ESO)** is a vendor-neutral fourth option.

| Option | Mechanism | Choose when |
|---|---|---|
| **External Secrets Operator** | Controller reads a store definition, fetches the keys listed in each `ExternalSecret`, creates/updates a Kubernetes Secret on a refresh interval | You want one pattern across backends; Secrets stay GitOps-friendly because Git holds only references |
| **Vault Secrets Operator** | HashiCorp operator syncing Vault secrets to Kubernetes Secrets via CRDs | Vault-only shop wanting vendor-native features such as dynamic secret leases |
| **Agent Injector** | Mutating webhook adds an agent sidecar that renders secrets into a shared in-memory volume | You want secrets as files that never become a Kubernetes Secret object |
| **CSI provider** | Secrets mounted as a volume through the Secrets Store CSI driver | You already use the CSI driver |

### External Secrets example

```yaml
apiVersion: external-secrets.io/v1          # check the API version your installed operator serves
kind: ClusterSecretStore
metadata: { name: vault }
spec:
  provider:
    vault:
      server: https://vault.acme.internal
      path: kv
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso
          serviceAccountRef: { name: external-secrets, namespace: external-secrets }
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: orders-api-secrets, namespace: orders-prod }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault, kind: ClusterSecretStore }
  target: { name: orders-api-secrets }
  data:
  - secretKey: ConnectionStrings__Orders          # becomes the environment variable name in the .NET app
    remoteRef: { key: orders/prod/orders-api/db, property: connectionString }
```

Operational points:

- A refreshed Kubernetes Secret does **not** restart pods that read it through environment variables. Use a reloader controller, or roll the Deployment, when values must change.
- Scope ESO's own Vault role narrowly: a `ClusterSecretStore` can read whatever its policy allows, so prefer per-namespace `SecretStore` resources with per-team roles in a multi-team cluster.
- Git holds only `remoteRef` paths, never values, so Argo CD can manage them safely.

## 9.7 Dynamic database credentials

The **database secrets engine** creates a PostgreSQL role per request, with a TTL, and revokes it afterwards. This removes shared long-lived passwords.

```bash
vault secrets enable database
vault write database/config/orders-pg \
  plugin_name=postgresql-database-plugin \
  allowed_roles="orders-api" \
  connection_url="postgresql://{{username}}:{{password}}@orders-pg-rw.databases:5432/orders?sslmode=require" \
  username="vault-admin" password="<rotate-me>"
vault write database/roles/orders-api \
  db_name=orders-pg \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT orders_app TO \"{{name}}\";" \
  default_ttl=1h max_ttl=24h
```

Applications (or VSO) then request credentials from `database/creds/orders-api`. Make sure connection pools and EF Core handle credential rotation (reconnect on auth failure, or pair short TTLs with a sidecar/operator that refreshes configuration). If that is too heavy at first, begin with static secrets in KV and a rotation schedule, and move to dynamic later.

## 9.8 PKI and internal certificates

Vault's PKI engine can act as your internal CA (root offline, intermediate in Vault). cert-manager's Vault issuer then issues workload and ingress certificates automatically, with short lifetimes and renewal. This also supplies certificates for internal services such as Kafka listeners and the registry.

## 9.9 Cleaning up leaked secrets

If a credential has ever been committed to Git:

1. **Rotate it first.** Removing it from the file does not remove it from history.
2. Move the new value into Vault; reference it from Helm values via `remoteRef`.
3. Add **secret detection** to CI and **push rules** blocking key and certificate files.
4. Keep development settings as placeholders; real values come from user-secrets or environment variables.
5. Optionally rewrite history, but only after rotation, and remember forks and caches may keep the old content.

## 9.10 Operating Vault

| Task | Practice |
|---|---|
| Backups | Scheduled Raft snapshots, copied off-site, restore tested |
| Upgrades | One node at a time, standby first, read the release notes |
| Monitoring | Telemetry endpoint to Prometheus: seal status, leader changes, request latency, token and lease counts |
| Access review | Quarterly review of policies and auth roles |
| Break-glass | Documented procedure with split custody of recovery material |
| Audit | Alert on policy changes, root token use, and failed auth bursts |

## Done when

- [ ] Vault HA with TLS, audit devices and tested auto-unseal.
- [ ] A pod receives a secret through External Secrets, scoped to its own path.
- [ ] A CI job fetches a credential via an ID token on a protected branch only.
- [ ] No plaintext secrets remain in Git; leaked ones have been rotated.

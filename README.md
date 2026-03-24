# homelab-gitops

ArgoCD app-of-apps GitOps repository for an OKD 4.19 Kubernetes cluster. Push to main, ArgoCD syncs to the cluster. No `kubectl apply` from a laptop. Ever.

This is the deployment layer of [Project Sentinel](https://github.com/koiakoia) — every Kubernetes workload defined in git, deployed by ArgoCD, with secrets pulled from HashiCorp Vault via External Secrets Operator.

## Architecture

```
Git push → ArgoCD detects → Syncs to OKD cluster
                              ↓
                    External Secrets Operator
                              ↓
                    HashiCorp Vault (secrets)
```

ArgoCD watches this repo. When you push, it reconciles the cluster state to match. Drift gets auto-corrected.

## What's Running

### Applications (`apps/`)

| App | What It Does |
|-----|-------------|
| **Keycloak** | SSO/OIDC provider — one login for all services |
| **Harbor** | Container registry with image signing (cosign) |
| **Grafana** | Dashboards and alerting |
| **ArgoCD** | GitOps deployment (self-managed) |
| **Backstage** | Developer portal and service catalog |
| **DefectDojo** | Vulnerability management |
| **NetBox** | DCIM/IPAM |
| **Matrix + Element** | Encrypted chat |
| **Homepage** | Service dashboard |
| **Jellyfin** | Media server |
| **Plane** | Project management |
| **Ntfy** | Push notifications |
| **Kyverno** | Policy enforcement (image signatures, registry allowlists) |
| **Falco** | Runtime security |
| **Istio** | Service mesh with STRICT mTLS |
| **Jaeger** | Distributed tracing |

### Service Mesh (`apps/mesh-config/`)
Istio with STRICT mTLS mesh-wide. All pod-to-pod traffic is encrypted. Gateway + VirtualServices for ingress routing.

### Backstage Catalog (`backstage-catalog/`)
Full service catalog with component definitions for every infrastructure service, API, and team.

### Cluster Config (`clusters/overwatch/`)
- App-of-apps root application
- Per-app Helm value overrides
- MachineConfigs for CoreOS nodes
- Monitoring stack (Grafana, Prometheus)
- Storage (NFS provisioner, iSCSI PVs)

## CI/CD Pipeline

```
yamllint → trivy-config → gitleaks → checkov
```

CI validates manifests. ArgoCD handles actual deployment — no CI-driven `kubectl apply`.

## Key Design Decisions

- **Air-gapped cluster** — OKD nodes have no direct internet. Images pulled through Harbor mirror.
- **Kyverno enforces signed images** — Every container must be cosign-signed from the internal Harbor registry.
- **External Secrets Operator** — No secrets in git. ESO pulls from Vault at deploy time.
- **Network policies per app** — Each app has explicit ingress/egress policies. Default deny.

## Getting Started

1. Copy `.env.example` to `.env` and configure your domain and IPs
2. Update `clusters/overwatch/root-app.yaml` with your ArgoCD repo URL
3. Apply the root app: `oc apply -f clusters/overwatch/root-app.yaml`
4. ArgoCD picks up everything else automatically

All domain names and IPs are `${VARIABLE}` placeholders — see `.env.example`.

## Related Repos

- [homelab-iac](https://github.com/koiakoia/homelab-iac) — Infrastructure provisioning (VMs, networking, Vault)
- [homelab-compliance](https://github.com/koiakoia/homelab-compliance) — NIST 800-53 compliance artifacts
- [homelab-platform](https://github.com/koiakoia/homelab-platform) — OKD cluster bootstrap

## License

Apache 2.0 — see [LICENSE](LICENSE).

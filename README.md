# Golden Path Platform

One golden path from a template repo to a running, observable service on Kubernetes —
with GitOps, supply-chain security, and self-service conventions.

**Status:** v0.1 in progress. Unchecked items below are not built yet.

## What v0.1 will demonstrate

- A developer creates a repo from a template and gets CI/CD, TLS, secrets, metrics,
  and rollback without touching the cluster.
- GitHub Actions builds, scans with Syft and Grype, signs with cosign, and updates
  the desired image digest.
- Argo CD reconciles staging and prod. Promotion is a PR; rollback is a Git revert.
- External Secrets pulls credentials from AWS SSM. No secrets in Git.
- Prometheus and Grafana provide golden-signal dashboards and one actionable alert.

## Architecture

```text
Developer
  └─ golden-path-template → GitHub Actions (test → Syft → Grype → cosign) → GHCR
        └─ digest PR → Argo CD → k3s on IONOS
              ├─ Traefik + cert-manager → notes.fahadkhan.me
              ├─ notes-staging / notes-prod
              ├─ Notes API + Postgres (ClusterIP)
              └─ Prometheus/Grafana + alerts

Terraform → S3 state, SSM, Cloudflare DNS
ESO → AWS SSM Parameter Store
```

## Repos

| Repo | Role |
|---|---|
| `platform-core` (this repo) | Terraform, cluster addons, Argo CD Applications, dashboards, docs |
| `golden-path-template` | Runnable Go scaffold, chart, CI, deploy values |
| `demo-service` | Generated from the template; full Notes API |

## Repo layout

```text
platform-core/
├── terraform/        # S3 state, SSM, Cloudflare DNS
├── bootstrap/        # Argo CD, ESO, cert-manager, monitoring
├── apps/             # Argo CD Application manifests
├── policies/         # v0.2: Kyverno
├── dashboards/       # Grafana dashboards
└── docs/             # spec, decisions, journal, runbooks
```

## Quickstart

1. Provision the AlmaLinux 9 VPS and install k3s.
2. Install cert-manager, ESO, Argo CD, and kube-prometheus-stack.
3. Create namespaces `notes-staging` and `notes-prod`.
4. Apply the Argo CD Applications in `apps/`.
5. Create a service from `golden-path-template` and push.

## What's implemented

- [ ] k3s cluster on IONOS
- [ ] Platform addons
- [ ] Walking-skeleton template
- [ ] Staging deploy via CI and Argo CD
- [ ] Prod promotion by digest
- [ ] TLS
- [ ] Secrets from SSM
- [ ] Dashboard and alert
- [ ] Rollback drill

## Design decisions

See [docs/decisions.md](docs/decisions.md). The spec is in [docs/spec.md](docs/spec.md).

## Known limitations

- Single VPS; no HA or disaster recovery.
- Flannel in v0.1; NetworkPolicy is not enforced until Calico in v0.2.
- Kyverno admission policy is v0.2.
- Chart and CI workflow are copied into app repos; shared chart and reusable workflow
  are v0.2.
- One Postgres per app namespace; no backups or HA.
- Manual Argo CD Application onboarding; ApplicationSet is v0.2.
- No auth on the demo API.
- No Loki or tracing.

## License

MIT

# Golden Path Platform - v0.1

## Problem
On a lean engineering team, every new repo meant manual server setup, a bespoke deploy script, and per-repo secrets wiring. I was the bottleneck; deploys were inconsistent; the team couldn't afford a paid platform. The correct path wasn't the easy path.

## Users
- Primary: application developers. First user: me.
- Secondary: on-call engineer (dashboards, runbook, rollback); security reviewer
  (signed images, SBOM, scan results, secret flow).

## Outcome
A developer can go from a template repo to a running staging service with HTTPS,
health checks, metrics, and rollback in under 30 minutes, without SSH or kubectl.
Promotion to prod is a PR; rollback is a Git revert.

## Scope
- Workload: Go Notes API + Postgres. Health, metrics, CRUD, pagination,
  synthetic owner_email PII. No auth.
- Cluster: k3s on IONOS VPS, AlmaLinux, 4 GB / 4 vCPU / 120 GB.
- Environments: notes-staging and notes-prod namespaces.
- CI: GitHub Actions; tests; Syft SBOM; Grype scan; cosign keyless sign/attest;
  GHCR; digest bump.
- CD: Argo CD; staging auto-sync/self-heal; prod manual sync; promotion by
  digest PR.
- Ingress/TLS: Traefik + cert-manager + Let's Encrypt;
  notes.example.com and staging.notes.example.com.
- Secrets: External Secrets Operator + AWS SSM Parameter Store.
- Data: Postgres per app namespace, ClusterIP only.
- Observability: trimmed kube-prometheus-stack; one dashboard
  (rate, errors, p95, saturation); one alert.
- Security: non-root, read-only filesystem, dropped capabilities; Grype + Syft +
  cosign; Flannel in v0.1, no NetworkPolicy claim; Kyverno v0.2.
- IaC: Terraform for S3 state, SSM, Cloudflare DNS. VPS prepared manually.
- Onboarding: manual Argo CD Application per service in v0.1; ApplicationSet v0.2.

## Architecture
Developer → golden-path-template → GitHub Actions
  (test → Syft → Grype → cosign) → GHCR
→ digest PR → Argo CD (platform-core) → k3s on IONOS
  ├─ Traefik + cert-manager → notes.example.com
  ├─ notes-staging / notes-prod namespaces
  ├─ Notes API + Postgres (ClusterIP)
  └─ Prometheus/Grafana + alerts
Terraform → S3 state, SSM, Cloudflare DNS
ESO → AWS SSM Parameter Store

## Acceptance criteria
- [ ] New repo from template deploys to staging by merging a PR, zero SSH/kubectl.
- [ ] Same image digest promoted to prod by PR; no rebuild.
- [ ] Rollback is one Git revert, under 5 minutes.
- [ ] CI fails on failing tests, critical Grype findings, missing SBOM/signature.
- [ ] Image signed keylessly; SBOM attested and verifiable.
- [ ] HTTPS enforced; HTTP redirects.
- [ ] Rolling update causes zero failed requests.
- [ ] Postgres ClusterIP-only; credentials from SSM, never Git.
- [ ] Grafana shows rate/errors/p95/saturation; alert at >5% 5xx for 5 minutes.
- [ ] Synthetic PII redacted from logs.
- [ ] A second engineer deploys from the README in under 30 minutes.
- [ ] Terraform destroy leaves no billable AWS resources.

## Out of scope
Multi-cloud, multi-region, HA, DR, backups, EKS, service mesh, Gateway API,
Backstage, Crossplane, custom operators, auth/SSO, arbitrary repos,
multi-tenancy, Loki/tracing, compliance certs, FinOps, NetworkPolicy/Kyverno
in v0.1.

## Risks / open questions
- 4 GB VPS may OOM with monitoring; trim or move Grafana to a free tier.
- AlmaLinux SELinux/firewalld need care; install the k3s policy; don't disable SELinux.
- Cloudflare DNS-only; change nameservers.
- Scope creep: auth, EKS, Backstage are v0.2+.

# Decisions

Lightweight ADRs. One per expensive-to-reverse choice.
Format: Context → Decision → Consequences.

## ADR-001: Kubernetes on a single IONOS VPS with k3s
**Status:** Accepted
**Context:** The goal is to learn platform engineering, not operate cloud infrastructure.
EKS adds control-plane cost and IAM/VPC complexity before the golden path exists.
**Decision:** One AlmaLinux 9 VPS, 4 GB RAM, 4 vCPU, k3s. EKS is a later timeboxed sprint.
**Consequences:** Cheap and fast; single point of failure; no managed control plane.
4 GB requires trimming the monitoring stack. Revisit in v0.3.

## ADR-002: GitOps with Argo CD; promotion by PR, rollback by Git revert
**Status:** Accepted
**Context:** CI could deploy directly with a production kubeconfig, but that puts cluster
credentials in GitHub and makes Git advisory rather than authoritative.
**Decision:** GitHub Actions builds, scans, signs, and updates desired state in Git.
Argo CD is the only deployer. Prod promotion changes the image digest via PR;
rollback reverts the commit.
**Consequences:** No production credentials in CI; Git is the audit trail; drift is
corrected. Prod sync is manual in v0.1.

## ADR-003: Promote image digests, never tags
**Status:** Accepted
**Context:** Tags are mutable; reusing a tag makes rollbacks non-deterministic.
**Decision:** CI writes the image digest into `deploy/<env>/values.yaml`; Argo CD
deploys the digest.
**Consequences:** Reproducible deploys and rollbacks; GHCR must retain old digests.

## ADR-004: Grype + Syft for supply-chain scanning; no Trivy
**Status:** Accepted
**Context:** Need image vulnerability scanning and an SBOM. Trivy is excluded for
supply-chain trust reasons.
**Decision:** Syft generates a CycloneDX SBOM; Grype scans it; CI fails on critical
findings, tightening to high with an ignore file for unfixable CVEs.
**Consequences:** Explicit SBOM artifact; two tools instead of one; no Trivy anywhere.

## ADR-005: cosign keyless signing with GitHub OIDC
**Status:** Accepted
**Context:** Need provenance and tamper evidence without managing long-lived keys.
**Decision:** Sign and attest the image digest in GitHub Actions using cosign keyless;
verify with pinned certificate identity and OIDC issuer.
**Consequences:** No signing keys to manage; signatures depend on GitHub OIDC;
Kyverno enforcement is v0.2.

## ADR-006: External Secrets Operator backed by AWS SSM
**Status:** Accepted
**Context:** No secrets in Git; the app needs a database password and future credentials.
**Decision:** ESO syncs SSM parameters into Kubernetes Secrets. Onboarding creates
the SSM parameter.
**Consequences:** AWS dependency for secrets; ESO controller credentials are a
bootstrap secret; document the gap.

## ADR-007: Cloudflare DNS-only for v0.1
**Status:** Accepted
**Context:** Cloudflare proxying terminates TLS at its edge and allows HTTP inspection;
Let's Encrypt HTTP-01 is simplest without proxying.
**Decision:** Cloudflare hosts DNS; records are DNS-only; TLS terminates at Traefik
with cert-manager.
**Consequences:** End-to-end TLS to origin; no WAF/CDN yet; proxying later requires
Full (Strict).

## ADR-008: Flannel in v0.1; Calico and Kyverno in v0.2
**Status:** Accepted
**Context:** Flannel does not enforce NetworkPolicy; Calico and Kyverno add complexity
and memory to a 4 GB box.
**Decision:** v0.1 uses Flannel and does not claim network isolation. Postgres is
ClusterIP-only. v0.2 installs Calico for NetworkPolicy and Kyverno for admission policy.
**Consequences:** Faster path to a working golden path; documented security gap;
v0.2 is a meaningful upgrade.

## ADR-009: One Postgres per app namespace in v0.1
**Status:** Accepted
**Context:** The template must deploy without platform-side database provisioning;
a shared DB or operator adds onboarding complexity.
**Decision:** The app chart runs a single Postgres instance in the app namespace,
ClusterIP-only. v0.2 considers CloudNativePG or a managed database.
**Consequences:** Self-contained template; not production-grade HA or backups;
documented limitation.

## ADR-010: Manual Argo CD Application onboarding in v0.1
**Status:** Accepted
**Context:** ApplicationSet automation is valuable but not required to prove the
golden path.
**Decision:** The platform team adds one Argo CD Application per service/environment,
plus namespace and secret path. v0.2 replaces this with an ApplicationSet.
**Consequences:** Simple and explicit; onboarding is a platform-team bottleneck;
v0.2 removes it.

## ADR-011: Terraform scope is AWS state, SSM, and DNS only
**Status:** Accepted
**Context:** The VPS is already provisioned; the IONOS Terraform provider targets
IONOS Cloud, not necessarily the VPS product. Managing Kubernetes addons with
Terraform creates state drift.
**Decision:** Terraform owns the S3 state backend, SSM parameters, and Cloudflare
DNS. Helm and Argo CD own cluster addons.
**Consequences:** Clean boundary; less Terraform surface; server provisioning is
manual or cloud-init.

# Kubernetes GitOps Platform

This repository contains the declarative state for a Kubernetes platform
managed by FluxCD. The reconciliation graph is split by responsibility:
repository sources, CRDs, controllers, controller-backed configuration,
observability, and application entrypoints.

The design favors small ownership boundaries, explicit dependency ordering,
health-gated controller rollout, and reproducible cluster state. Platform
controllers are installed before dependent resources, secrets are never stored
in plaintext, and ingress/authentication behavior is modeled directly in
Kubernetes resources.

## Architecture

```text
infrastructure-sources
  -> infrastructure-crds
    -> infrastructure-controllers
      -> infrastructure-config
        -> apps

infrastructure-controllers
  -> observability
```

Flux owns the platform baseline. Argo CD is installed as a downstream
application delivery plane for application repositories, with infrastructure
ownership kept separate to avoid reconciliation conflicts.

## Core Platform Capabilities

### GitOps Control Plane

- FluxCD reconciles the cluster from `clusters/kubeadm`.
- Reconciliation is split into dependency-aware Kustomizations.
- Helm releases are managed declaratively with remediation and drift detection.
- Controller readiness gates later reconciliation layers.
- SOPS with AWS KMS protects committed Kubernetes Secrets.

### Networking And Ingress

- Cilium is managed through Flux with kube-proxy replacement enabled.
- Gateway API is the primary ingress abstraction.
- A shared `networking` namespace owns Gateway routes and TLS-facing resources.
- cert-manager issues wildcard certificates through Cloudflare DNS-01.
- HTTPRoutes are kept as discrete manifests so route ownership, backend
  references, and authentication paths are reviewable per service.
- Cross-namespace routing is controlled with explicit ReferenceGrants.

### Identity And Access

- A central OIDC provider backs human authentication.
- Group membership is used as the coarse-grained authorization boundary.
- Pomerium protects services that do not provide suitable native OIDC.
- Argo CD, Grafana, Harbor, and the Kubernetes API use native OIDC where that
  improves CLI/API compatibility.
- Kubernetes API authentication is OIDC-enabled and mapped through RBAC.

### Secrets Management

- SOPS remains the bootstrap secret mechanism for Flux.
- Bitwarden Secrets Manager Operator is installed through Flux.
- A Bitwarden machine-account token is stored as a SOPS-encrypted Kubernetes
  Secret.
- `BitwardenSecret` resources sync selected Bitwarden secrets into normal
  Kubernetes Secrets.
- Runtime secrets can be migrated out of Git incrementally without changing the
  Kubernetes Secret consumption model.

Bootstrap secrets remain SOPS-encrypted. Runtime secrets can be sourced from
Bitwarden and projected into the namespace where the workload expects them.

### Observability

- kube-prometheus-stack provides Prometheus, Alertmanager, Grafana, and core
  Kubernetes monitoring.
- Prometheus uses persistent storage with retention period and retention-size
  limits.
- Grafana uses native OIDC and includes Prometheus, Alertmanager, and Loki
  datasources.
- Loki stores logs with 7-day retention.
- Alloy collects Kubernetes pod logs and forwards them to Loki.
- Cilium and Hubble metrics are enabled and selected by Prometheus
  ServiceMonitors.

### Storage And Registry

- Local Path Provisioner is tracked in Flux and configured as the default
  storage class with volume expansion enabled.
- SeaweedFS provides an S3-compatible endpoint with both path-style and
  virtual-hosted bucket access.
- Harbor provides a private container registry with native Pocket ID OIDC.
- Harbor has separate `public` and `private` projects:
  - `public` supports anonymous pulls.
  - `private` requires authenticated Docker/Helm access.
- Harbor uses Trivy scanning, scan-on-push, and daily scheduled scan-all jobs.

## Repository Layout

```text
clusters/
  kubeadm/                 Flux entrypoint and reconciliation graph

infrastructure/
  sources/                 HelmRepository and source definitions
  crds/                    Explicit CRD layer
  controllers/             Platform controllers and stateful services
  config/                  Cluster-level config consumed by controllers

observability/
  prometheus-stack/        Metrics, Grafana, Alertmanager
  loki/                    Log storage
  alloy/                   Log collection

apps/
  base/                    Application base entrypoint
  production/              Production app entrypoint
```

## Operating Model

- HelmRepository resources live under `infrastructure/sources`.
- Controllers and stateful platform services live under
  `infrastructure/controllers`.
- Controller-backed configuration lives under `infrastructure/config`.
- Health checks are added when a controller must be ready before dependent
  resources apply.
- Secrets are either SOPS-encrypted or synchronized from Bitwarden.
- Flux-owned and Argo-owned resources are kept separate.
- Native OIDC is preferred for services with CLI/API workflows; Pomerium is used
  for browser-gated internal tools.

## Current State

The platform currently includes:

- FluxCD GitOps reconciliation
- SOPS/AWS KMS secret encryption
- Bitwarden Secrets Manager sync
- Cilium networking and Gateway API
- cert-manager with DNS-01 wildcard certificates
- Pomerium OIDC edge auth
- Argo CD for future app-of-apps deployment
- kube-prometheus-stack, Grafana, Loki, Alloy, and Hubble observability
- SeaweedFS S3-compatible object storage
- Harbor registry with OIDC, public/private projects, and Trivy scanning
- Reloader for opt-in workload restarts on Secret/ConfigMap changes

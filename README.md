# Kubernetes GitOps Platform

> **Archived repository**
>
> This repository is a snapshot of how I designed and managed my Kubernetes
> platform using FluxCD.
>
> The active production configuration is being moved to a separate repository.
> This repository is retained primarily as a technical reference and portfolio
> artifact to demonstrate my practical experience with Kubernetes, GitOps,
> networking, identity, observability, secrets management, storage, and
> platform operations.
>
> It should therefore be treated as an architectural snapshot rather than the
> current production source of truth.

## Overview

This repository contains the declarative state of a Kubernetes platform managed
through FluxCD.

The reconciliation graph is separated by responsibility:

```text
infrastructure-sources
  -> infrastructure-crds
    -> infrastructure-controllers
      -> infrastructure-config
        -> apps

infrastructure-controllers
  -> observability
```

The platform was designed around:

* Small and explicit ownership boundaries
* Dependency-aware reconciliation
* Health-gated controller rollout
* Reproducible cluster state
* GitOps-managed Helm releases
* Encrypted secret bootstrap
* OIDC-based identity
* Gateway API-based ingress
* Integrated metrics, logging, and network observability
* Clear separation between platform and application delivery

Flux owns the Kubernetes platform baseline.

Argo CD is installed as a downstream application delivery plane for application
repositories, while infrastructure ownership remains with Flux to avoid
reconciliation conflicts.

## Core Platform Capabilities

### GitOps Control Plane

* FluxCD reconciles the cluster from `clusters/kubeadm`.
* Reconciliation is split into dependency-aware Kustomizations.
* Helm releases are managed declaratively with remediation and drift detection.
* Controller readiness gates later reconciliation layers.
* SOPS with AWS KMS protects committed Kubernetes Secrets.
* Platform configuration is reconstructed from Git rather than maintained
  manually on the cluster.

### Networking and Ingress

* Cilium provides cluster networking with kube-proxy replacement enabled.
* Gateway API is used as the primary ingress abstraction.
* A shared `networking` namespace owns Gateway and TLS-facing resources.
* cert-manager issues wildcard certificates through Cloudflare DNS-01.
* HTTPRoutes are maintained as discrete resources so backend routing,
  authentication, and ownership remain reviewable.
* Cross-namespace routing is controlled through explicit ReferenceGrants.
* Cilium and Hubble provide network-level visibility and metrics.

### Identity and Access

* A central OIDC provider backs human authentication.
* Group membership provides the primary coarse-grained authorization boundary.
* Pomerium protects browser-facing services without suitable native OIDC.
* Argo CD, Grafana, Harbor, and the Kubernetes API use native OIDC where CLI or
  API compatibility benefits from it.
* Kubernetes API authentication is OIDC-enabled and mapped to Kubernetes RBAC.

The general preference is:

```text
Native OIDC
    ↓
Use when the application supports it well

Pomerium
    ↓
Use for browser-gated internal services
```

### Secrets Management

SOPS remains the bootstrap secret mechanism for Flux.

* Secrets committed to Git are encrypted using SOPS and AWS KMS.
* Bitwarden Secrets Manager Operator is installed through Flux.
* The Bitwarden machine-account credential is itself stored as a
  SOPS-encrypted Kubernetes Secret.
* `BitwardenSecret` resources synchronize selected secrets into normal
  Kubernetes Secrets.
* Applications continue consuming standard Kubernetes Secrets regardless of
  whether their source is SOPS or Bitwarden.

This allows runtime credentials to be migrated out of Git incrementally without
requiring applications to change their Secret consumption model.

```text
Bootstrap secrets
    -> SOPS + AWS KMS

Runtime secrets
    -> Bitwarden Secrets Manager
    -> BitwardenSecret
    -> Kubernetes Secret
    -> Workload
```

### Observability

The observability stack includes:

* kube-prometheus-stack
* Prometheus
* Alertmanager
* Grafana
* Loki
* Alloy
* Cilium/Hubble metrics

Prometheus uses persistent storage with both retention-duration and
retention-size limits.

Grafana includes datasources for:

* Prometheus
* Alertmanager
* Loki

Loki provides centralized log storage with 7-day retention.

Alloy collects Kubernetes workload logs and forwards them to Loki.

Cilium and Hubble metrics are exposed through ServiceMonitors and collected by
Prometheus.

### Storage and Registry

Local Path Provisioner provides the cluster's default dynamic storage class and
supports volume expansion.

SeaweedFS provides S3-compatible object storage with support for:

* Path-style bucket access
* Virtual-hosted bucket access

Harbor provides the private container registry and includes:

* Native OIDC authentication
* `public` and `private` projects
* Anonymous pulls from the public project
* Authenticated Docker and Helm access for private artifacts
* Trivy vulnerability scanning
* Scan-on-push
* Scheduled daily scan-all jobs

## Repository Layout

```text
clusters/
  kubeadm/
    Flux entrypoint and reconciliation graph

infrastructure/
  sources/
    HelmRepository and source definitions

  crds/
    Explicit CRD installation layer

  controllers/
    Platform controllers and stateful platform services

  config/
    Cluster-level resources consumed by controllers

observability/
  prometheus-stack/
    Prometheus, Grafana, Alertmanager and Kubernetes monitoring

  loki/
    Centralized log storage

  alloy/
    Kubernetes log collection

apps/
  base/
    Application base entrypoint

  production/
    Production application entrypoint from this repository snapshot
```

## Reconciliation Model

A typical dependency path looks like:

```text
HelmRepository
      ↓
CRDs
      ↓
Controller
      ↓
Controller health check
      ↓
Cluster configuration
      ↓
Application resources
```

This prevents resources from being applied before the controllers responsible
for them are ready.

For example:

```text
cert-manager CRDs
      ↓
cert-manager
      ↓
ClusterIssuer
      ↓
Certificate
```

The same model is applied across other controller-backed resources where
ordering matters.

## Operating Principles

The repository reflects several conventions I use when operating Kubernetes
with Flux:

* HelmRepository resources belong under `infrastructure/sources`.
* CRDs are separated when explicit installation ordering is useful.
* Controllers and stateful platform services belong under
  `infrastructure/controllers`.
* Controller-backed configuration belongs under `infrastructure/config`.
* Health checks are added when dependent reconciliation must wait for
  controller readiness.
* Plaintext Kubernetes Secrets are never committed.
* Bootstrap secrets are SOPS-encrypted.
* Runtime secrets can be synchronized from Bitwarden.
* Flux-owned and Argo-owned resources have clearly separated ownership.
* Native OIDC is preferred when applications expose CLI/API workflows.
* Pomerium is used where browser-level access control is more appropriate.
* Routes, authentication policies, certificates, and cross-namespace access are
  modeled explicitly in Kubernetes rather than configured manually.

## Platform Components

This snapshot demonstrates configuration and integration of:

* FluxCD
* Kubernetes Kustomizations
* HelmRelease
* SOPS
* AWS KMS
* Bitwarden Secrets Manager Operator
* Cilium
* Hubble
* Gateway API
* cert-manager
* Cloudflare DNS-01
* Pomerium
* OIDC authentication
* Kubernetes RBAC
* Argo CD
* kube-prometheus-stack
* Prometheus
* Alertmanager
* Grafana
* Loki
* Alloy
* Local Path Provisioner
* SeaweedFS
* Harbor
* Trivy
* Reloader

## Repository Status

This repository is **archived**.

It represents one iteration of my Kubernetes platform and the GitOps patterns I
used to operate it.

The production environment is moving to a separate source of truth, so this
repository will no longer receive updates intended to track the live cluster.

I am keeping it publicly available as part of my CV/portfolio so that the
implementation can be inspected rather than simply listing technologies or
certifications.

The repository is intended to demonstrate practical experience with:

* Kubernetes platform engineering
* FluxCD and GitOps reconciliation
* Dependency and controller lifecycle management
* Kubernetes networking with Cilium
* Gateway API
* TLS and DNS automation
* OIDC and RBAC
* Secrets management
* Prometheus-based monitoring
* Centralized logging
* Container registry operations
* Kubernetes storage
* Declarative infrastructure design

It is **not intended to be deployed verbatim as a generic Kubernetes
distribution or production reference architecture**. Some decisions are
specific to the cluster, infrastructure, operational preferences, and
constraints under which this platform was built.

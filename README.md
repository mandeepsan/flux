# Flux Cluster Configuration

This repository manages the `kubeadm` Kubernetes cluster with Flux.

The cluster entrypoint is `clusters/kubeadm`. Flux reconciliation is split into
small dependency layers so that cluster foundations become ready before
dependent configuration and workloads are applied.

## Reconciliation Layers

```text
infrastructure-sources
  -> infrastructure-crds
    -> infrastructure-controllers
      -> infrastructure-config
        -> apps

infrastructure-controllers
  -> observability
```

Use `dependsOn`, `wait`, and targeted health checks when adding resources that
install controllers, CRDs, or workloads consumed by later layers.

# OpenShift Service Mesh 3.3: Sidecar and Ambient Mode Walkthrough

This guide covers the installation of Red Hat OpenShift Service Mesh 3.3 (via the Sail Operator), Kiali, and demonstrates both traditional Sidecar injection and the new Ambient mesh architecture with the bookinfo application from the Istio project.

A **service mesh** is an infrastructure layer that transparently adds **encryption (mTLS)**, **observability (metrics, traces)**, and **traffic control (routing, authorization)** to service-to-service communication — without modifying application code.

OpenShift Service Mesh 3.3 offers two data-plane architectures:

| | **Sidecar Mode** | **Ambient Mode** |
|---|---|---|
| Proxy location | One Envoy per pod (injected container) | Shared ztunnel (per-node) + optional waypoint (per-namespace) |
| Resource overhead | Higher (memory/CPU per pod) | Lower (shared infrastructure) |
| L7 policy | Always available | Only when waypoint is deployed |
| Pod restart required | Yes (to inject sidecar) | No (transparent enrollment) |

## Guide Structure

| Step | File | Description |
|------|------|-------------|
| 0 | [00-setup.md](./00-setup.md) | Prerequisites, operator installation, `istioctl`, Kiali, monitoring configuration |
| A | [01-sidecar-mode.md](./01-sidecar-mode.md) | Traditional sidecar injection: CNI, control plane, bookinfo deployment, mTLS, ingress gateway, authorization policies |
| B | [02-ambient-mode.md](./02-ambient-mode.md) | Ambient mesh: OVN-Kubernetes prerequisites, ztunnel, waypoint proxies, Kubernetes Gateway API, mTLS |

## Quick Start

1. Complete the **[Setup](./00-setup.md)** (operators, Kiali, monitoring)
2. Choose your path:
   - **[Sidecar Mode (A)](./01-sidecar-mode.md)** — one Envoy per pod, full L7 from the start
   - **[Ambient Mode (B)](./02-ambient-mode.md)** — shared ztunnel per node, no sidecar injection needed

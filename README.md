# Kubernetes AI Conformance Platform

This repository contains the documentation, manifests, examples, and evidence required for **CNCF Kubernetes AI Conformance** certification.

## Overview

A Kubernetes-based AI/MLOps platform designed for:

- **Training**: Distributed GPU-accelerated training with gang scheduling
- **Inference**: Model serving with advanced traffic management via Gateway API
- **Agentic Workloads**: Support for complex AI pipelines and operators

## Hardware Environment

| Node | GPU | Role |
|------|-----|------|
| Node 1 | NVIDIA RTX 5090 | GPU Worker |
| Node 2 | NVIDIA RTX 4090 | GPU Worker |
| Node 3 | NVIDIA RTX 6000 Pro | GPU Worker |

**Kubernetes Version**: v1.34.1

## Components

| Component | Version | Purpose |
|-----------|---------|---------|
| NVIDIA Device Plugin | v0.18.1 | GPU resource management |
| Kueue | 0.15.2 | Gang scheduling for distributed workloads |
| KubeRay Operator | v1.5.1 | Ray cluster orchestration |
| Gateway API | v1.1.0 | Advanced traffic routing |
| Prometheus + DCGM Exporter | latest | GPU metrics and monitoring |

## Quick Start

### Prerequisites

See [docs/prerequisites.md](docs/prerequisites.md) for complete requirements.

### Installation

```bash
# 1. Install NVIDIA Device Plugin
kubectl apply -f manifests/nvidia-device-plugin/nvidia-device-plugin.yaml

# 2. Install Volcano Scheduler
kubectl apply -f manifests/volcano/

# 3. Install Gateway API
kubectl apply -f manifests/gateway-api/

# 4. Install KubeRay Operator
kubectl apply -f manifests/ray/

# 5. Install Prometheus Monitoring
kubectl apply -f manifests/prometheus/
```

### Validation

```bash
# Verify GPU resources
kubectl get nodes -o json | jq '.items[].status.allocatable["nvidia.com/gpu"]'

# Test GPU workload
kubectl apply -f manifests/nvidia-device-plugin/gpu-test-pod.yaml
kubectl logs gpu-test
```

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | Platform architecture and conformance statement |
| [Prerequisites](docs/prerequisites.md) | Environment requirements |
| [Installation](docs/install.md) | Step-by-step installation guide |
| [GPU Support](docs/gpu.md) | GPU setup and validation |
| [Scheduling](docs/scheduling.md) | Advanced scheduling features |
| [Gateway API](docs/gateway-api.md) | Inference traffic management |
| [Monitoring](docs/monitoring.md) | Metrics and observability |
| [Conformance](docs/conformance.md) | Conformance checklist |

## Examples

- [Ray GPU Job](examples/ray-gpu-job/) - Distributed GPU training example
- [Ray Serve](examples/ray-serve/) - Model serving with Ray
- [Inference Gateway](examples/inference-gateway/) - Gateway API routing example

## Conformance Status

| Requirement | Status |
|-------------|--------|
| GPU Enablement | ✅ Pass |
| AI/MLOps Platform | ✅ Pass |
| Gang Scheduling | ✅ Pass |
| Gateway API | ✅ Pass |
| Accelerator Metrics | ✅ Pass |
| Dynamic Resource Allocation | ✅ Pass |
| Cluster Autoscaling | ✅ Pass |
| HPA for AI Workloads |  ✅ Pass |

## License

Apache License 2.0 - see [LICENSE](LICENSE) for details.

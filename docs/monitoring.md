# Monitoring and Metrics Support

This document describes the prerequisites, installation, and validation steps for monitoring and metrics collection in a Kubernetes cluster to satisfy CNCF Kubernetes AI Conformance requirements.

## Overview

Monitoring in AI workloads ensures observability of:

- GPU utilization and memory usage
- Pod resource usage
- AI operator health (e.g., Ray, Kubeflow)
- Horizontal and vertical scaling performance
- Custom metrics from AI inference services

Common tools:

- Prometheus for metrics collection
- Grafana for visualization
- Node Exporter / GPU metrics exporters
- OpenTelemetry for standardized metrics

## Prerequisites

- Kubernetes 1.27+
- Prometheus Operator installed or Prometheus deployed
- GPU nodes with NVIDIA drivers and device plugin installed (see `docs/gpu.md`)

## Prometheus Installation Example

Using kube-prometheus-stack Helm chart:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

Verify Prometheus Pods:

```
kubectl get pods -n monitoring
```

## GPU Metrics Exporter

Deploy NVIDIA DCGM Exporter for GPU metrics:

```
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/deployments/k8s/dcgm-exporter-daemonset.yaml
```

Verify metrics endpoints:

```
kubectl get pods -n monitoring -l app=dcgm-exporter
```

Prometheus will scrape GPU metrics from `/metrics` endpoint on each node.

## AI Workload Metrics

For Ray or Kubeflow workloads:

- Ensure custom metrics endpoints are exposed
- Prometheus `ServiceMonitor` can be used:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ray-servicemonitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: ray
  endpoints:
    - port: metrics
      interval: 30s
```

## Validation

1. Check that Prometheus targets include GPU nodes and AI workloads:

```
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prometheus-prometheus 9090:9090
```

1. Open `http://localhost:9090/targets` to verify scraping.
2. Query GPU metrics:

```
nvidia_smi_memory_used_bytes
nvidia_smi_utilization_gpu
```

1. Ensure AI operators (e.g., Ray, Kubeflow) metrics are visible and accurate.

## Conformance Notes

- Monitoring must include metrics for GPU, AI workloads, and custom metrics relevant to AI/MLOps workloads.
- If monitoring cannot be deployed in the environment, checklist items may be marked as `N/A` in `docs/conformance.md` with justification.
- Collected metrics should follow standard formats (Prometheus, OpenTelemetry) for interoperability.

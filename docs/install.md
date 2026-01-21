# Installation Guide

This document provides **step-by-step installation instructions** to deploy the Kubernetes AI Conformance platform in a clean Kubernetes cluster that satisfies the prerequisites defined in `docs/prerequisites.md`.

All steps are designed to be **fully reproducible** by CNCF reviewers.

---

## 1. Pre-flight Checks

Before installation, verify the following:

```bash
kubectl version --short
kubectl get nodes -o wide
```

Confirm that:

* Kubernetes version is **v1.35.x**
* At least one node reports GPU capacity

Verify GPU visibility on the node:

```bash
nvidia-smi
```

---

## 2. Install NVIDIA Device Plugin

The NVIDIA Device Plugin exposes GPUs as Kubernetes resources.

### 2.1 Install

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/refs/tags/v0.18.1/deployments/static/nvidia-device-plugin.yml
```

### 2.2 Verify

```bash
kubectl get pods -n kube-system | grep nvidia
kubectl describe node <gpu-node> | grep -A5 Allocatable
```

Expected output includes:

```
nvidia.com/gpu: <number>
```

---

## 3. Install Gang Scheduler (Volcano)

Volcano provides gang scheduling required for distributed AI workloads.

### 3.1 Install Volcano

```bash
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml
```

### 3.2 Verify

```bash
kubectl get pods -n volcano-system
kubectl get scheduler
```

Ensure all Volcano components are in **Running** state.

---

## 4. Install Gateway API

Gateway API is required for advanced inference traffic management.

### 4.1 Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

### 4.2 Verify

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

---

## 5. Install Prometheus (Monitoring)

Prometheus is used to collect platform and accelerator metrics.

### 5.1 Install Prometheus

```bash
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
```

### 5.2 Verify

```bash
kubectl get pods -n monitoring
```

---

## 6. Install GPU Metrics Exporter

The NVIDIA DCGM Exporter exposes GPU metrics in Prometheus format.

### 6.1 Install DCGM Exporter

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/dcgm-exporter/main/deployment/dcgm-exporter.yaml
```

### 6.2 Verify

```bash
kubectl get pods -n gpu-monitoring
```

---

## 7. Install KubeRay Operator (AI/MLOps Platform)

KubeRay provides CRDs and controllers for distributed Ray workloads.

### 7.1 Install KubeRay Operator

```bash
kubectl apply -f https://raw.githubusercontent.com/ray-project/kuberay/release-1.1.0/manifests/base/kuberay-operator.yaml
```

### 7.2 Verify

```bash
kubectl get pods -n ray-system
kubectl get crd | grep ray.io
```

---

## 8. (Optional) Enable Dynamic Resource Allocation (DRA)

If using DRA-based workflows, ensure the feature gates are enabled on the API server and kubelet.

Verify DRA support:

```bash
kubectl api-resources | grep resourceclaims
```

---

## 9. Validation Checklist

After installation, confirm the following:

* GPU resources are allocatable
* Volcano scheduler is active
* Gateway API CRDs are present
* Prometheus is scraping targets
* KubeRay Operator is running

```bash
kubectl get pods -A
```

---

## 10. Next Steps

Proceed to:

* `docs/conformance.md` for checklist mapping
* `examples/` to deploy sample AI workloads

These steps complete the platform installation required for Kubernetes AI Conformance validation.

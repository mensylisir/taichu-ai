# Prerequisites

This document describes the **prerequisites required to reproduce and validate** the Kubernetes AI Conformance platform described in this repository.

The goal is to ensure that **CNCF reviewers can prepare an equivalent environment** and successfully follow the installation and validation steps without ambiguity.

---

## 1. Kubernetes Cluster Requirements

### 1.1 Kubernetes Version

* Kubernetes **v1.35.x** is required
* The cluster must support:

  * CustomResourceDefinitions (CRDs)
  * Admission webhooks
  * Gateway API (v1 or later)
  * Dynamic Resource Allocation (DRA)

> Earlier Kubernetes versions are **not supported** for this conformance validation.

---

### 1.2 Cluster Type

The platform is **Kubernetes distribution agnostic** and has been validated on:

* kubeadm-based clusters
* Bare-metal Kubernetes clusters
* Cloud-based Kubernetes clusters (with GPU support)

The following are **explicitly out of scope**:

* k3s
* MicroK8s
* Single-node development clusters

---

## 2. Node Requirements

### 2.1 Control Plane Nodes

* Minimum: **1 control plane node**
* Recommended: **3 control plane nodes** for HA validation

Requirements:

* CPU: 4 cores or more
* Memory: 8 GiB or more
* Disk: 50 GiB or more

---

### 2.2 Worker Nodes (GPU Nodes)

At least **one GPU-capable worker node** is required.

#### Hardware Requirements

* NVIDIA GPU (physical)
* CUDA-capable device
* PCIe passthrough or bare-metal access

#### Software Requirements

* Linux kernel with NVIDIA driver support
* NVIDIA GPU driver installed on the host

> GPU simulation or emulation is **not sufficient** for conformance validation.

---

## 3. Operating System Requirements

Supported operating systems include:

* Linux x86_64
* Linux ARM64

Validated distributions include (but are not limited to):

* Ubuntu 20.04 / 22.04
* RHEL / Rocky Linux / AlmaLinux
* Other CNCF-compatible Linux distributions

---

## 4. Container Runtime

A **CRI-compliant container runtime** is required.

Validated runtimes:

* containerd (recommended)
* CRI-O

Docker Engine (dockershim) is **not supported**.

---

## 5. GPU Software Stack

### 5.1 NVIDIA Driver

* NVIDIA driver must be installed on GPU nodes
* Driver version must be compatible with the installed CUDA toolkit

Verification command:

```bash
nvidia-smi
```

---

### 5.2 NVIDIA Container Toolkit

The NVIDIA Container Toolkit must be installed to enable GPU access inside containers.

Verification:

```bash
ctr version
```

---

### 5.3 NVIDIA Device Plugin

* The platform requires **nvidia-device-plugin**
* GPUs must be exposed as Kubernetes resources (`nvidia.com/gpu`)

Verification:

```bash
kubectl get nodes -o json | jq '.items[].status.allocatable'
```

---

## 6. Networking Requirements

### 6.1 CNI Plugin

A Kubernetes-compatible CNI plugin is required.

Validated CNIs include:

* Calico
* Cilium
* Flannel

The CNI must support:

* Pod-to-Pod networking
* NetworkPolicy (optional but recommended)

---

### 6.2 Gateway API Support

The cluster must support **Kubernetes Gateway API**:

* Gateway API CRDs installed
* Support for HTTPRoute resources

Ingress-only environments are **not sufficient**.

---

## 7. Storage Requirements

* PersistentVolume support is required
* Any CSI-compatible storage provider is acceptable

Use cases include:

* Model artifacts
* Training checkpoints
* Logs

---

## 8. Scheduling & Resource Management

The cluster must allow installation of:

* A gang scheduling solution:

  * Volcano **or** Kueue
* Dynamic Resource Allocation (DRA)

The default Kubernetes scheduler **alone is not sufficient**.

---

## 9. Observability Requirements

The cluster must allow installation of:

* Prometheus
* GPU metrics exporters (e.g., DCGM Exporter)

The environment must permit:

* Metrics scraping
* Custom metrics discovery

---

## 10. Access & Permissions

The installer must have:

* `cluster-admin` privileges
* Ability to install CRDs and cluster-scoped resources

---

## 11. Out-of-Scope Capabilities

The following capabilities are **explicitly out of scope** for this platform and may be marked as **N/A** in the conformance checklist:

* Cluster Autoscaler
* GPU node pool auto-scaling
* Horizontal Pod Autoscaler for AI workloads
* Service mesh integration

---

## 12. Summary

If all prerequisites described in this document are satisfied, CNCF reviewers should be able to:

* Install all required components
* Run GPU-accelerated AI workloads
* Validate Kubernetes AI Conformance requirements

No proprietary systems or hosted services are required to complete the validation.

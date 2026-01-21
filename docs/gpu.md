# GPU Support

This document describes the GPU prerequisites, installation steps, and validation required for a Kubernetes cluster to satisfy the GPU-related requirements of the CNCF Kubernetes AI Conformance.

## Hardware and Driver Requirements

* At least one node equipped with an NVIDIA GPU
* Supported GPU models include (examples):

  * NVIDIA A100 / V100 / T4 / L4 / RTX series
* Supported operating systems on GPU nodes:

  * Ubuntu 20.04 / 22.04
  * UOS / Kylin (with validated NVIDIA driver support)

### NVIDIA Driver

Install the official NVIDIA driver on each GPU node. After installation, verify with:

```bash
nvidia-smi
```

The command should display the GPU model and driver version without errors.

## CUDA and Container Runtime

* CUDA version: >= 11.x (must be compatible with the installed driver)
* Container runtime:

  * containerd

### NVIDIA Container Toolkit

Install the NVIDIA Container Toolkit on GPU nodes:

```bash
apt-get install -y nvidia-container-toolkit
```

Configure containerd to use the NVIDIA runtime:

```bash
nvidia-ctk runtime configure --runtime=containerd
systemctl restart containerd
```

## Kubernetes GPU Components

### NVIDIA Device Plugin

Deploy the official NVIDIA device plugin using a DaemonSet:

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/refs/tags/v0.18.1/deployments/static/nvidia-device-plugin.yml
```

After deployment, verify that GPU resources are advertised:

```bash
kubectl get nodes -o json | jq '.items[].status.capacity["nvidia.com/gpu"]'
```

## Validation

Create a test Pod that requests a GPU and runs `nvidia-smi` inside the container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  containers:
  - name: cuda
    image: nvidia/cuda:12.2.0-base-ubuntu22.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
```

If the Pod runs successfully and outputs GPU information, GPU support is correctly configured.

## Conformance Notes

* GPU support is required only for AI workloads that explicitly request accelerator resources.
* If GPU hardware is not available in a given environment, GPU-related checklist items may be marked as `N/A` in `docs/conformance.md` with justification.

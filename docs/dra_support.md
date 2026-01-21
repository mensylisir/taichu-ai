# DRA v1 APIs are enabled in 1.34 by default

In AI training and inference scenarios, when multiple applications need to share GPU resources, Kubernetes DRA API can be used to break through the scheduling limitations of traditional device plugins, enabling dynamic GPU allocation and fine-grained resource control between Pods, thereby improving GPU utilization and reducing costs.

## Prerequisite

Create cluster with Kubernetes version >= 1.34.
Helm version 3 or above installed.

## Verify the environment

1. Command

```bash
kubectl api-resources --api-group=resource.k8s.io -o wide
```

2. Expected output

```bash
NAME                     SHORTNAMES   APIVERSION           NAMESPACED   KIND                    VERBS                                                        CATEGORIES
deviceclasses                         resource.k8s.io/v1   false        DeviceClass             create,delete,deletecollection,get,list,patch,update,watch
resourceclaims                        resource.k8s.io/v1   true         ResourceClaim           create,delete,deletecollection,get,list,patch,update,watch
resourceclaimtemplates                resource.k8s.io/v1   true         ResourceClaimTemplate   create,delete,deletecollection,get,list,patch,update,watch
resourceslices                        resource.k8s.io/v1   false        ResourceSlice           create,delete,deletecollection,get,list,patch,update,watch
```

## Install NVIDIA GPU Operator

1. Install NVIDIA GPU Operator using Helm

```bash
helm install gpu-operator gpu-operator --repo https://helm.ngc.nvidia.com/nvidia --version v25.3.3 -n gpu-operator --create-namespace
```

2. Verify the installation

```bash
(base) root@ai18:~/wode/gpu-operator# kubectl -n gpu-operator  get pod
NAME                                                          READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-4cg7l                                   1/1     Running     0          54m
gpu-feature-discovery-hl89d                                   1/1     Running     0          54m
gpu-operator-5b95b8d948-jqkzq                                 1/1     Running     0          88m
gpu-operator-node-feature-discovery-gc-664cfcb6d9-9nnmn       1/1     Running     0          88m
gpu-operator-node-feature-discovery-master-6f97d7f876-f7pzj   1/1     Running     0          88m
gpu-operator-node-feature-discovery-worker-4sdj7              1/1     Running     0          88m
gpu-operator-node-feature-discovery-worker-wtdnv              1/1     Running     0          88m
nvidia-container-toolkit-daemonset-fxhdg                      1/1     Running     0          54m
nvidia-container-toolkit-daemonset-sdjvq                      1/1     Running     0          54m
nvidia-cuda-validator-gkgzh                                   0/1     Completed   0          49m
nvidia-cuda-validator-rgxdt                                   0/1     Completed   0          19m
nvidia-dcgm-exporter-6w9dv                                  ·  1/1     Running     0          54m
nvidia-dcgm-exporter-jq2jx                                    1/1     Running     0          54m
nvidia-device-plugin-daemonset-bw66f                          1/1     Running     0          54m
nvidia-device-plugin-daemonset-jhmls                          1/1     Running     0          54m
nvidia-operator-validator-sz5ch                               1/1     Running     0          54m
nvidia-operator-validator-tbfj8                               1/1     Running     0          54m
(base) root@ai18:~/wode/gpu-operator# 

```

3. Create configmap

```
kubectl apply -n gpu-operator -f - <<eof
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: gpu-operator
data:
  config.yaml: |-
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4
eof
```

4. Patch the clusterpolicy

```
kubectl patch clusterpolicy cluster-policy \
    --type merge \
    -p '{"spec": {"devicePlugin": {"config": {"name": "device-plugin-config", "default": "config.yaml"}}}}'
```

5. Verify node resources

```
Capacity:
  cpu:                20
  ephemeral-storage:  958827632Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             65579888Ki
  nvidia.com/gpu:     4
  pods:               110
Allocatable:
  cpu:                20
  ephemeral-storage:  883655544189
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             65272688Ki
  nvidia.com/gpu:     4
  pods:               110

```

## Install NVIDIA DRA driver
```bash
helm install nvidia-dra-driver-gpu oci://ghcr.io/nvidia/k8s-dra-driver-gpu --version 25.8.0-dev-13a73595-chart -n k8s-dra-driver-gpu --create-namespace
```

```bash
(base) root@ai18:~/wode# kubectl -n k8s-dra-driver-gpu get pod
NAME                                                READY   STATUS    RESTARTS   AGE
nvidia-dra-driver-gpu-controller-789ffdc44d-clltm   1/1     Running   0          64m
nvidia-dra-driver-gpu-kubelet-plugin-2tfj9          2/2     Running   0          64m
nvidia-dra-driver-gpu-kubelet-plugin-ww8nh          2/2     Running   0          64m
(base) root@ai18:~/wode# 
```

## Create a resource claim template
```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    devices:
      requests:
      - name: single-gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          allocationMode: ExactCount
          count: 1
```

## Create a deployment that requests the GPU resource
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-cuda-check
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pytorch-cuda-check
  template:
    metadata:
      labels:
        app: pytorch-cuda-check
    spec:
      runtimeClassName: nvidia
      containers:
        - name: pytorch-cuda-check
          image: nvcr.io/nvidia/pytorch:25.09-py3
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                python3 -c "import torch; print(torch.cuda.device_count())"
                sleep 30
              done
          resources:
            claims:
              - name: single-gpu
      resourceClaims:
        - name: single-gpu
          resourceClaimTemplateName: gpu-claim-template
```

## Ensure that access to accelerators from within containers is properly isolated.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeName: ai18
  containers:
  - name: cuda
    image: nvidia/cuda:12.0-base
    command: ["sleep", "3600"]
    resourceClaims:
    - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimName: claim-a
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-gpu-pod
spec:
  nodeName: ai18
  containers:
  - name: alpine
    image: alpine
    command: ["sleep", "3600"]
```

```
kubectl exec gpu-pod -- nvidia-smi -L
kubectl exec no-gpu-pod -- ls /dev/nvidia0
```

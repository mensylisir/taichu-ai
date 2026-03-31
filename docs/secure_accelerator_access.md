# secure_accelerator_access

## Prerequests
Create cluster with Kubernetes version >= 1.34

## Test1

Deploy a Pod to a node with available accelerators, without requesting accelerator resources in the Pod spec. Execute a command in the Pod to probe for accelerator devices, and the command should fail or report that no accelerator devices are found.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: gpu-test
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: docker.io/nginx:alpine-perl
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 20
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 5
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: "64"
            memory: 10Gi
          requests:
            cpu: "64"
            memory: 10Gi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: app
        operator: Equal
        value: gpu
```

Verify pods

```
(base) root@ai18:~/wode/ssa# kubectl get pod
NAME                       READY   STATUS    RESTARTS   AGE
gpu-test-99bb5ff46-dwxnd   1/1     Running   0          23s
(base) root@ai18:~/wode/ssa# 
```

execute nvidia-smi in container：

```
(base) root@ai18:~/wode/ssa# kubectl exec -it gpu-test-99bb5ff46-dwxnd -- sh
/ # nvidia-smi
sh: nvidia-smi: not found
/ # 

```

## Test2

Create two Pods, each is allocated an accelerator resource. Execute a command in one Pod to attempt to access the other Pod’s accelerator, and should be denied.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: nvidia-mig-48gb-template
  namespace: default
spec:
  spec:
    devices:
      requests:
      - name: myclaim
        exactly:
          deviceClassName: mig.nvidia.com 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-test-dra
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-dra-app
  template:
    metadata:
      labels:
        app: gpu-dra-app
    spec:
      nodeSelector:
        kubernetes.io/hostname: ps
      tolerations:
      - effect: NoSchedule
        key: app
        operator: Equal
        value: gpu
      containers:
      - name: pytorch-container
        image: nvcr.io/nvidia/pytorch:25.09-py3
        imagePullPolicy: IfNotPresent
        command: ["/bin/bash", "-c", "--"]
        args: ["nvidia-smi -L && python3 -c 'import torch; print(torch.cuda.get_device_properties(0))' && while true; do sleep 30; done;"]
        resources:
          requests:
            cpu: "2"
            memory: 5Gi
          claims:
          - name: gpu-resource
      resourceClaims:
      - name: gpu-resource
        resourceClaimTemplateName: nvidia-mig-48gb-template
```

two pods have been created on same node

```
(base) root@ai18:~/wode/ssa# kubectl get pod -owide
NAME                            READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
gpu-test-dra-66dcc797f7-88khs   1/1     Running   0          62s   172.20.56.207   ps     <none>           <none>
gpu-test-dra-66dcc797f7-brxjd   1/1     Running   0          62s   172.20.56.206   ps     <none>           <none>
(base) root@ai18:~/wode/ssa# 
```

check gpu index of pod gpu-test-dra-66dcc797f7-88khs

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test-dra-66dcc797f7-88khs -- nvidia-smi -L
GPU 0: NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition (UUID: GPU-267e97f4-9a65-8884-83ea-cfb6b7211c1e)
  MIG 2g.48gb     Device  0: (UUID: MIG-693536c1-6e29-5e30-b461-3c8e1ae1df67)
```

check gpu index of pod gpu-test-dra-66dcc797f7-brxjd

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test-dra-66dcc797f7-brxjd -- nvidia-smi -L
GPU 0: NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition (UUID: GPU-3fc81e18-7342-dc82-8748-4b8a1c2b104d)
  MIG 2g.48gb     Device  0: (UUID: MIG-0a577d74-f9ce-5ad8-8d39-a7b7eb7915c5) 
```

exec nvidia-smi in pod gpu-test-dra-66dcc797f7-88khs

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test-dra-66dcc797f7-88khs  -- nvidia-smi
Mon Mar 30 09:32:54 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.153.02             Driver Version: 570.153.02     CUDA Version: 13.0     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:98:00.0 Off |                   On |
| 30%   29C    P8              7W /  300W |                  N/A   |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |                     Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                       BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  0    2   0   0  |             128MiB / 48512MiB    | 94      0 |  2   2    2    0    2 |
|                  |                 0MiB / 65535MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
(base) root@ai18:~/wode/ssa# 
```

exec nvidia-smi in pod gpu-test-dra-66dcc797f7-brxjd

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test-dra-66dcc797f7-brxjd  -- nvidia-smi
Mon Mar 30 09:33:25 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.153.02             Driver Version: 570.153.02     CUDA Version: 13.0     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    On  |   00000000:5A:00.0 Off |                   On |
| 30%   33C    P8              6W /  300W |                  N/A   |     N/A      Default |
|                                         |                        |              Enabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| MIG devices:                                                                            |
+------------------+----------------------------------+-----------+-----------------------+
| GPU  GI  CI  MIG |                     Memory-Usage |        Vol|        Shared         |
|      ID  ID  Dev |                       BAR1-Usage | SM     Unc| CE ENC  DEC  OFA  JPG |
|                  |                                  |        ECC|                       |
|==================+==================================+===========+=======================|
|  0    2   0   0  |             128MiB / 48512MiB    | 94      0 |  2   2    2    0    2 |
|                  |                 0MiB / 65535MiB  |           |                       |
+------------------+----------------------------------+-----------+-----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
(base) root@ai18:~/wode/ssa# 
```

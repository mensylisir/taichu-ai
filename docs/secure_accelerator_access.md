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
apiVersion: apps/v1
kind: Deployment
metadata:
  generation: 10
  labels:
    app: nginx
  name: gpu-test2
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nvcr.io/nvidia/pytorch:25.09-py3
        imagePullPolicy: Always
        name: nginx
        command: ["/bin/bash", "-c", "--"]
        args: ["while true; do sleep 30; done;"]
        resources:
          limits:
            cpu: "4"
            memory: 10Gi
            nvidia.com/gpu: "2"
          requests:
            cpu: "4"
            memory: 10Gi
            nvidia.com/gpu: "2"
      nodeSelector:
        kubernetes.io/hostname: ai18
      restartPolicy: Always
      tolerations:
      - effect: NoSchedule
        key: app
        operator: Equal
        value: gpu
```

two pods have been created on same node

```
(base) root@ai18:~/wode/ssa# kubectl get pod -owide
NAME                         READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
gpu-test2-5475b7d7d5-54g87   1/1     Running   0          71s   172.20.121.139   ai18   <none>           <none>
gpu-test2-5475b7d7d5-g9j26   1/1     Running   0          71s   172.20.121.140   ai18   <none>           <none>
(base) root@ai18:~/wode/ssa# 

```

check gpu index of pod gpu-test2-5475b7d7d5-54g87

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test2-5475b7d7d5-54g87 -- env | grep -i NVIDIA_VISIBLE_DEVICES
NVIDIA_VISIBLE_DEVICES=GPU-20bf8b16-5485-68a3-1c91-568ced47bf03
(base) root@ai18:~/wode/ssa# 

```

check gpu index of pod gpu-test2-5475b7d7d5-g9j26

```
(base) root@ai18:~/wode/ssa#  kubectl exec gpu-test2-5475b7d7d5-g9j26 -- env | grep -i NVIDIA_VISIBLE_DEVICES
NVIDIA_VISIBLE_DEVICES=GPU-20bf8b16-5485-68a3-1c91-568ced47bf03
(base) root@ai18:~/wode/ssa# 

```

exec nvidia-smi in pod gpu-test2-5475b7d7d5-54g87

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test2-5475b7d7d5-54g87  -- nvidia-smi
Wed Jan 21 08:42:05 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 5090 D      On  |   00000000:01:00.0 Off |                  N/A |
|  0%   36C    P8             20W /  600W |      41MiB /  32607MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
(base) root@ai18:~/wode/ssa# 
```

exec nvidia-smi in pod gpu-test2-5475b7d7d5-g9j26

```
(base) root@ai18:~/wode/ssa# kubectl exec gpu-test2-5475b7d7d5-g9j26  -- nvidia-smi
Wed Jan 21 08:42:54 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 5090 D      On  |   00000000:01:00.0 Off |                  N/A |
|  0%   36C    P8             23W /  600W |      41MiB /  32607MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
(base) root@ai18:~/wode/ssa# 

```

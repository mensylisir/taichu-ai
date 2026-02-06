# cluster_autoscaling

## Prerequisites

Create cluster with Kubernetes version >= 1.34.
Helm version 3 or above installed.

## Create GPU MachineDeployment and Enable Cluster Autoscaler

### Create GPU MachineDeployment

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  annotations:
    capacity.cluster-autoscaler.kubernetes.io/cpu: "20"
    capacity.cluster-autoscaler.kubernetes.io/gpu-count: "1"
    capacity.cluster-autoscaler.kubernetes.io/gpu-type: nvidia.com/gpu
    capacity.cluster-autoscaler.kubernetes.io/labels: purpose=gpu
    capacity.cluster-autoscaler.kubernetes.io/memory: 60Gi
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size: "2"
    cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size: "0"
  labels:
    cluster-autoscaler-enabled: "true"
    cluster.x-k8s.io/cluster-name: my-cluster
  name: my-cluster-gpu
  namespace: default
  ownerReferences:
  - apiVersion: cluster.x-k8s.io/v1beta2
    kind: Cluster
    name: my-cluster
spec:
  clusterName: my-cluster
  replicas: 0
  rollout:
    strategy:
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0
      type: RollingUpdate
  selector:
    matchLabels:
      cluster.x-k8s.io/cluster-name: my-cluster
  template:
    metadata:
      labels:
        cluster-autoscaler-enabled: "true"
        cluster.x-k8s.io/cluster-name: my-cluster
        purpose: gpu
    spec:
      bootstrap:
        configRef:
          apiGroup: infrastructure.cluster.x-k8s.io
          kind: BootstrapKubeconfig
          name: my-cluster-workers-bootstrap-kubeconfig
      clusterName: my-cluster
      infrastructureRef:
        apiGroup: infrastructure.cluster.x-k8s.io
        kind: ByoMachineTemplate
        name: my-cluster-gpu-tmpl
      version: v1.34.1
```

### Create ByoMachineTemplate

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: ByoMachineTemplate
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: my-cluster
  name: my-cluster-gpu-tmpl
  namespace: default
  ownerReferences:
  - apiVersion: cluster.x-k8s.io/v1beta2
    kind: Cluster
    name: my-cluster
spec:
  capacity:
    cpu: "20"
    gpuCount: "1"
    gpuType: nvidia.com/gpu
    labels: purpose=gpu
    memory: 60Gi
  template:
    spec:
      bootstrapConfigRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: BootstrapKubeconfig
        name: my-cluster-workers-bootstrap-kubeconfig
      installerRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: K8sInstallerConfigTemplate
        name: my-cluster-worker-installer
        namespace: default
      joinMode: tlsBootstrap
      selector:
        matchLabels:
          purpose: gpu
```

### Install Cluster Autoscaler

```yaml
# myvalues.yaml
cloudProvider: clusterapi

image:
  repository: registry.k8s.io/autoscaling/cluster-autoscaler
  tag: v1.34.2
  pullPolicy: IfNotPresent

autoDiscovery:
  clusterName: my-cluster
  labels:
    - cluster-autoscaler-enabled: "true"

extraArgs:
  v: 4
  stderrthreshold: info
  logtostderr: true
  node-group-auto-discovery: "clusterapi:clusterName=my-cluster"
  expander: random
  scale-down-enabled: true
  scale-down-unneeded-time: 5m
  scan-interval: 10s
  kube-client-qps: 40
  kube-client-burst: 80

rbac:
  create: true
  clusterScoped: true
  additionalRules:
    - apiGroups:
        - infrastructure.cluster.x-k8s.io
      resources:
        - byoclusters
        - byoclustertemplates
        - byohosts
        - byomachines
        - byomachinetemplates
      verbs:
        - get
        - list
        - watch
    - apiGroups: ["cluster.x-k8s.io"]
      resources: ["clusters", "machinedeployments", "machinesets", "machines", "machinepools"]
      verbs: ["get", "list", "watch", "update", "patch"]

nodeSelector:
  node-role.kubernetes.io/control-plane: ""

tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"

clusterAPIMode: incluster-incluster

```

Execute the following command to install the Cluster Autoscaler:

``` shell
helm install cluster-autoscaler autoscaler/cluster-autoscaler -f myvalues.yaml -n kube-system
```

## Configure GPU Resources and Node Affinity for Applications

### Write workload YAML with proper node affinity, toleration, and GPU resource configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
  labels:
    app: pytorch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pytorch
  template:
    metadata:
      labels:
        app: pytorch
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: purpose
                operator: In
                values:
                - gpu
      restartPolicy: Always
      containers:
        - args:
          - while true; do sleep 30; done;
          command:
          - /bin/bash
          - -c
          - --
          name: pytorch
          image: nvcr.io/nvidia/pytorch:25.09-py3
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "2"
              memory: 5Gi
              nvidia.com/gpu: 1
            requests:
              cpu: "2"
              memory: 5Gi
              nvidia.com/gpu: 1
```

### View Pod Events

```
(base) root@ai18:~/wode/ca-example# kubectl describe pod gpu-workload-665764584d-lkdbq 
Name:             gpu-workload-665764584d-lkdbq
Namespace:        default
Priority:         0
Service Account:  default
Node:             ai20/172.30.1.20
Start Time:       Fri, 06 Feb 2026 10:30:10 +0800
Labels:           app=pytorch
                  pod-template-hash=665764584d
Annotations:      <none>
Status:           Running
IP:               172.20.91.202
IPs:
  IP:           172.20.91.202
Controlled By:  ReplicaSet/gpu-workload-665764584d
Containers:
  pytorch:
    Container ID:  containerd://280e990d4cb3c15841fe5914418d8d74db0bb98ef95a8c3e265e80d781b49caf
    Image:         nvcr.io/nvidia/pytorch:25.09-py3
    Image ID:      easzlab.io.local:5000/nvidia/pytorch@sha256:6b2f480b0ad6b270998d2a2a2fde4aabcca6391408befad396d7fe7bc563e98e
    Port:          80/TCP
    Host Port:     0/TCP
    Command:
      /bin/bash
      -c
      --
    Args:
      while true; do sleep 30; done;
    State:          Running
      Started:      Fri, 06 Feb 2026 10:30:13 +0800
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:             2
      memory:          5Gi
      nvidia.com/gpu:  1
    Requests:
      cpu:             2
      memory:          5Gi
      nvidia.com/gpu:  1
    Environment:       <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-pn9t5 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-pn9t5:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason             Age                From                Message
  ----     ------             ----               ----                -------
  Warning  FailedScheduling   87s                default-scheduler   0/2 nodes are available: 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
  Normal   TriggeredScaleUp   52s                cluster-autoscaler  pod triggered scale-up: [{MachineDeployment/default/my-cluster-gpu 0->1 (max: 1)}]
  Warning  FailedScheduling   41s                default-scheduler   0/3 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }, 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
  Warning  FailedScheduling   22s (x2 over 39s)  default-scheduler   0/3 nodes are available: 1 Insufficient nvidia.com/gpu, 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
  Normal   NotTriggerScaleUp  22s                cluster-autoscaler  pod didn't trigger scale-up: 1 node(s) didn't match Pod's node affinity/selector, 1 max node group size reached
  Normal   Scheduled          10s                default-scheduler   Successfully assigned default/gpu-workload-665764584d-lkdbq to ai20
  Normal   Pulling            9s                 kubelet             Pulling image "nvcr.io/nvidia/pytorch:25.09-py3"
  Normal   Pulled             7s                 kubelet             Successfully pulled image "nvcr.io/nvidia/pytorch:25.09-py3" in 2.369s (2.369s including waiting). Image size: 8568718769 bytes.
  Normal   Created            7s                 kubelet             Created container: pytorch
  Normal   Started            7s                 kubelet             Started container pytorch
```

### After scale-up, confirm pods are running normally

```
(base) root@ai18:~/wode/ca-example# kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
gpu-workload-665764584d-lkdbq   1/1     Running   0          94s
(base) root@ai18:~/wode/ca-example# 
```

### Verify node status is normal and matches the expected instance type

```
(base) root@ai18:~/wode/ca-example# kubectl get node -l purpose=gpu
NAME   STATUS   ROLES    AGE     VERSION
ai20   Ready    <none>   3m36s   v1.34.1
```

### Verify scale-down by reducing replica count to 0

```
kubectl scale deploy gpu-workload --replicas=0
```

### Query the node list again

```
(base) root@ai18:~/wode/ca-example# kubectget node -l purpose=gpu
No resources found
(base) root@ai18:~/wode/ca-example# 
```

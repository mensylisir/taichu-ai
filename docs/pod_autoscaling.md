# Autoscale GPU Workloads Using DCGM Metrics and KEDA

The article describes how to autoscale GPU workloads by using GPU metrics collected by the DCGM exporter.These metrics are exposed through Prometheus and consumed by Kubernetes Event-Driven Autoscaling (KEDA) to automatically scale workloads based on real-time GPU utilization.

## Prerequisites

Create cluster with Kubernetes version >= 1.34.
Helm version 3 or above installed.

## Install Prometheus Operator

1. Install prometheus-operator helm chart
```
helm install prometheus oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack --namespace monitoring --create-namespace
```
2. Verify
```
(base) root@ai18:~# kubectl -n monitoring get pod
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          4m51s
prometheus-grafana-66f97d664b-jf5zb                      3/3     Running   0          6m26s
prometheus-kube-prometheus-operator-74888c95f7-52xdr     1/1     Running   0          6m27s
prometheus-kube-state-metrics-857895cb8d-sh7f2           1/1     Running   0          6m27s
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          4m50s
prometheus-prometheus-node-exporter-g42p8                1/1     Running   0          6m27s
prometheus-prometheus-node-exporter-q5sc2                1/1     Running   0          6m27s
```

## Install KEDA

KEDA can be deployed using YAML manifests, Helm charts, or Operator Hub. This article uses Helm charts.

1. Install keda helm chart
```
helm install keda kedacore/keda --namespace keda-system --create-namespace
```
2. Verify
```
(base) root@ai18:~/wode/keda# kubectl -n keda-system get pod
NAME                                               READY   STATUS    RESTARTS      AGE
keda-admission-webhooks-66d8d7cf95-lh68m           1/1     Running   0             2m3s
keda-operator-7d6d994857-62dxh                     1/1     Running   1 (77s ago)   2m3s
keda-operator-metrics-apiserver-5978cc84df-sg6fj   1/1     Running   0             2m3s
(base) root@ai18:~/wode/keda#
```

## Install DCGM Exporter

1. Install dcgm-exporter helm chart
```
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter --namespace dcgm-system --create-namespace
```
2. Verify
```
(base) root@ai18:~/wode/dcgm-exporter# kubectl -n dcgm-system get pod
NAME                  READY   STATUS    RESTARTS   AGE
dcgm-exporter-2b42f   1/1     Running   0          2m
dcgm-exporter-d242c   1/1     Running   0          66s
```

## Create ServiceMonitor

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter-sm
  namespace: monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: dcgm-exporter
  namespaceSelector:
    matchNames:
      - dcgm-system
  endpoints:
  - port: metrics
    interval: 15s
```

## Create Workload

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-test
  namespace: ha-test-namespace
  labels:
    taichu.io/app-name: hpa-test
spec:
  replicas: 1
  progressDeadlineSeconds: 9600
  selector:
    matchLabels:
      app: hpa-test
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    metadata:
      annotations:
        prometheus.io/port: "8000"
        prometheus.io/scrape: "true"
      labels:
        app: hpa-test
    spec:
      containers:
      - name: custom-container
        image: docker.io/vllm/vllm-openai:v0.11.0
        imagePullPolicy: Always
        command:
        - /bin/sh
        - -c
        - python3 -m vllm.entrypoints.openai.api_server --port 8000 --model /vllm-workspace/models/qwen/Qwen3-8B
          --tensor-parallel-size 1 --pipeline-parallel-size 1 --gpu-memory-utilization
          0.95 --max_model_len 4096 --served-model-name Qwen3-8B --trust-remote-code
        env:
        - name: VLLM_NODE
          value: "1"
        - name: AIHC_RESOURCE_FROM
          value: aihcpom
        - name: AIHC_APP
          value: hpa-test
        - name: AIHC_X_Region
          value: bj
        ports:
        - name: port-8000
          containerPort: 8000
          protocol: TCP
        resources:
          limits:
            nvidia.com/gpu: "1"
            cpu: "4"
            memory: 16Gi
        readinessProbe:
          tcpSocket:
            port: 8000
          initialDelaySeconds: 0
          periodSeconds: 5
          timeoutSeconds: 1
          successThreshold: 1
          failureThreshold: 6
        lifecycle:
          preStop:
            exec:
              command:
              - sleep
              - "30"
        securityContext:
          capabilities:
            add:
            - IPC_LOCK
        volumeMounts:
        - name: model-volume
          mountPath: /vllm-workspace/models
      terminationGracePeriodSeconds: 30
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      volumes: 
      - name: model-volume
        hostPath:
          path: /vllm-workspace/models 
          type: Directory
```

## Create ScaledObject

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-gpu-scaler
  namespace: ha-test-namespace
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-test
  minReplicaCount: 1
  maxReplicaCount: 2
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
        scaleUp:
          stabilizationWindowSeconds: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090 
      query: avg(DCGM_FI_DEV_GPU_UTIL{namespace="ha-test-namespace", pod=~"hpa-test-.*"})
      threshold: "12"
      metricName: gpu_util_pct
```

## Stress Testing and Autoscaling Verification

1. Execute GPU Stress Test in Pod
   First, get the name of the currently running pod:

   ```
    (base) root@ai18:~/wode/hpa-example# kubectl get pods -n ha-test-namespace -owide
    NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
    hpa-test-7d468b4969-7fgnj   1/1     Running   0          13m   172.20.91.240   ai20   <none>           <none>
    (base) root@ai18:~/wode/hpa-example# 
   ```

   Enter the Pod and execute GPU stress test to simulate high load scenarios:

   ```
    kubectl exec -it hpa-test-7d468b4969-7fgnj -n ha-test-namespace -- /bin/bash
    
    python3 << 'EOF'
    import torch
    import time
    
    device = torch.device("cuda:0")
    print("GPU stress test is RUNNING... Do not stop it with Ctrl+C!")
    
    while True:
        a = torch.randn(10000, 10000, device=device)
        b = torch.randn(10000, 10000, device=device)
        c = torch.matmul(a, b)
        torch.cuda.synchronize()
        time.sleep(0.1)
    EOF
   ```

2. Observe the HPA Status Change

HPA Status Change Process:

```
(base) root@ai18:~/wode/gateway-example# kubectl -n ha-test-namespace get hpa keda-hpa-vllm-gpu-scaler  -w
NAME                       REFERENCE             TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   0/12 (avg)   1         2         1          29s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   36/12 (avg)   1         2         1          46s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   9/12 (avg)    1         2         2          61s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   5500m/12 (avg)   1         2         2          76s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   21/12 (avg)      1         2         2          106s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   9/12 (avg)       1         2         2          2m16s
```

View detailed HPA information and scale-up events:

```
(base) root@ai18:~/wode/gateway-example# kubectl -n ha-test-namespace describe hpa keda-hpa-vllm-gpu-scaler 
Name:                                      keda-hpa-vllm-gpu-scaler
Namespace:                                 ha-test-namespace
Labels:                                    app.kubernetes.io/managed-by=keda-operator
                                           app.kubernetes.io/name=keda-hpa-vllm-gpu-scaler
                                           app.kubernetes.io/part-of=vllm-gpu-scaler
                                           app.kubernetes.io/version=2.18.3
                                           scaledobject.keda.sh/name=vllm-gpu-scaler
Annotations:                               <none>
CreationTimestamp:                         Tue, 20 Jan 2026 09:45:09 +0800
Reference:                                 Deployment/hpa-test
Metrics:                                   ( current / target )
  "s0-prometheus" (target average value):  9250m / 12
Min replicas:                              1
Max replicas:                              2
Behavior:
  Scale Up:
    Stabilization Window: 10 seconds
    Select Policy: Max
    Policies:
      - Type: Pods     Value: 4    Period: 15 seconds
      - Type: Percent  Value: 100  Period: 15 seconds
  Scale Down:
    Stabilization Window: 300 seconds
    Select Policy: Max
    Policies:
      - Type: Percent  Value: 100  Period: 15 seconds
Deployment pods:       2 current / 2 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: vllm-gpu-scaler,},MatchExpressions:[]LabelSelectorRequirement{},})
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  2m57s  horizontal-pod-autoscaler  New size: 2; reason: external metric s0-prometheus(&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: vllm-gpu-scaler,},MatchExpressions:[]LabelSelectorRequirement{},}) above target
(base) root@ai18:~/wode/gateway-example# 

```

3. Verify Pod Scale-up status

Check the deployment replica count changes:

```
(base) root@ai18:~/wode/hpa-example# kubectl get pods -n ha-test-namespace -owide -w
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
hpa-test-7d468b4969-7fgnj   1/1     Running   0          15m   172.20.91.240   ai20   <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Pending   0          1s    <none>          <none>   <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Pending   0          1s    <none>          ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     ContainerCreating   0          1s    <none>          ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Running             0          2s    172.20.121.175   ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   1/1     Running             0          78s   172.20.121.175   ai18     <none>           <none>
```

4. Scale-down Verification

After stopping the stress test, wait a moment for the HPA to scale down the pods.

```
(base) root@ai18:~/wode/gateway-example# kubectl -n ha-test-namespace get hpa keda-hpa-vllm-gpu-scaler  -w
NAME                       REFERENCE             TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   4500m/12 (avg)   1         2         2          8m38s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   2500m/12 (avg)   1         2         2          8m47s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   0/12 (avg)       1         2         2          9m17s
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   0/12 (avg)       1         2         2          13m
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   0/12 (avg)       1         2         1          13m
keda-hpa-vllm-gpu-scaler   Deployment/hpa-test   0/12 (avg)       1         2         1          14m

(base) root@ai18:~/wode/hpa-example# kubectl get pods -n ha-test-namespace -owide -w
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
hpa-test-7d468b4969-7fgnj   1/1     Running   0          15m   172.20.91.240   ai20   <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Pending   0          1s    <none>          <none>   <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Pending   0          1s    <none>          ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     ContainerCreating   0          1s    <none>          ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   0/1     Running             0          2s    172.20.121.175   ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   1/1     Running             0          78s   172.20.121.175   ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   1/1     Terminating         0          12m   172.20.121.175   ai18     <none>           <none>
hpa-test-7d468b4969-6bkns   1/1     Terminating         0          12m   172.20.121.175   ai18     <none>           <none>

```

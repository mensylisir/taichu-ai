# Scheduling Support

This document describes the prerequisites, installation steps, and validation required for Kubernetes scheduling features.

## Prerequisites

Create cluster with Kubernetes version >= 1.34.
Helm version 3 or above installed.

## Install Kueue with Helm
While Kueue comes with a robust set of default policies, it is not a 'one-size-fits-all' configuration; features like TopologyAwareScheduling remain optional and must be toggled via the Helm chart or Feature Gates. Additionally, framework support for MPI, Ray, and JobSet is not automatic—it requires manual activation.

1. Create and save a myvalues.yaml file to customize Kueue configuration
```
controllerManager:
  featureGates:
    - name: TopologyAwareScheduling
      enabled: true
    - name: LocalQueueMetrics
      enabled: true
  managerConfig:
    controllerManagerConfigYaml: |
      apiVersion: config.kueue.x-k8s.io/v1beta1
      kind: Configuration
      integrations:
        frameworks:
          - batch/job
          - kubeflow.org/mpijob
          - ray.io/rayjob
          - ray.io/raycluster
          - jobset.x-k8s.io/jobset
          - kubeflow.org/paddlejob
          - kubeflow.org/pytorchjob
          - kubeflow.org/tfjob
          - kubeflow.org/xgboostjob
          - kubeflow.org/jaxjob
```
2. Install the latest release version of the Kueue controller and CRDs.
```
helm install kueue oci://registry.k8s.io/kueue/charts/kueue \
  --version=0.15.2 \
  --namespace  kueue-system \
  --create-namespace \
  --values myvalues.yaml \
  --wait --timeout 300s
```

## Confirm deployment status

1. Confirm the installation of Kueue resources
```
(base) root@ai18:~/wode/kueue# kubectl get crds | grep kueue
admissionchecks.kueue.x-k8s.io                        2026-01-16T01:48:12Z
clusterqueues.kueue.x-k8s.io                          2026-01-16T01:48:12Z
cohorts.kueue.x-k8s.io                                2026-01-16T01:48:12Z
localqueues.kueue.x-k8s.io                            2026-01-16T01:48:12Z
multikueueclusters.kueue.x-k8s.io                     2026-01-16T01:48:12Z
multikueueconfigs.kueue.x-k8s.io                      2026-01-16T01:48:12Z
provisioningrequestconfigs.kueue.x-k8s.io             2026-01-16T01:48:12Z
resourceflavors.kueue.x-k8s.io                        2026-01-16T01:48:12Z
topologies.kueue.x-k8s.io                             2026-01-16T01:48:12Z
workloadpriorityclasses.kueue.x-k8s.io                2026-01-16T01:48:12Z
workloads.kueue.x-k8s.io                              2026-01-16T01:48:12Z
(base) root@ai18:~/wode/kueue# 
```

2. Confirm that controller pods are running properly.
```
(base) root@ai18:~/wode/kueue# kubectl -n kueue-system get pod -w
NAME                                       READY   STATUS    RESTARTS   AGE
kueue-controller-manager-c8bcd547d-cp6nb   1/1     Running   0          25s
```

## Define a ResourceFlavor object

1. Create and save a ResourceFlavor
```
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
 name: on-demand
```

2. Verify
```
(base) root@ai18:~/wode/kueue-example# kubectl get resourceflavors
NAME        AGE
on-demand   5s
```

## Create a ClusterQueue

1. Create a Kueue ClusterQueue
```
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
   name: "cluster-queue"
spec:
  namespaceSelector: {}
  resourceGroups:
   - coveredResources: ["cpu", "memory", "pods"]
     flavors:
       - name: on-demand
         resources:
           - name: "cpu"
             nominalQuota: 4
           - name: "memory"
             nominalQuota: 8Gi
           - name: "pods"
             nominalQuota: 5
```

2. Verify the ClusterQueue` manifest was applied
```
(base) root@ai18:~/wode/kueue-example# kubectl get clusterqueues
NAME            COHORT   PENDING WORKLOADS
cluster-queue            0
```

## Create a LocalQueue

1. Create a Kueue LocalQueue
```
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  name: sample-queue
spec:
  clusterQueue: cluster-queue
```

2. Verify the LocalQueue` manifest was applied
```
(base) root@ai18:~/wode/kueue-example# kubectl get localqueues
NAME           CLUSTERQUEUE   PENDING WORKLOADS   ADMITTED WORKLOADS
sample-queue   cluster-queue    0                   0
```

## Create 2 batch jobs

1.  Create two sample batch jobs
```
apiVersion: batch/v1
kind: Job
metadata:
  name: test-batch-1
  labels:
    kueue.x-k8s.io/queue-name: sample-queue
spec:
  parallelism: 1
  completions: 1
  template:
    spec:
      containers:
        - name: dummy-job
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          command: ["sh", "-c", "echo Running test-batch-1; sleep 60"]
          resources:
            requests:
              cpu: "1"
              memory: "500Mi"
            limits:
              cpu: "1"
              memory: "500Mi"
      restartPolicy: Never
---
apiVersion: batch/v1
kind: Job
metadata:
  name: test-batch-2
  labels:
    kueue.x-k8s.io/queue-name: sample-queue
spec:
  parallelism: 1
  completions: 1
  template:
    spec:
      containers:
        - name: dummy-job
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          command: ["sh", "-c", "echo Waiting in queue for CPUs...; sleep 30"]
          resources:
            requests:
              cpu: "2"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "1Gi"
      restartPolicy: Never
```
2. View the status of the batched workloads using the kubectl get command.
```
(base) root@ai18:~/wode/kueue-example# kubectl get workloads
NAME                     QUEUE          RESERVED IN     ADMITTED   FINISHED   AGE
job-test-batch-1-622ed   sample-queue   cluster-queue   True                  5s
job-test-batch-2-75a72   sample-queue   cluster-queue   True                  5s
(base) root@ai18:~/wode/kueue-example# 
```

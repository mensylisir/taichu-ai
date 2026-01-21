# Kubernetes AI Conformance Checklist

This document provides the checklist for CNCF Kubernetes AI Conformance. Each item corresponds to a requirement in the conformance program. Items that cannot be implemented in the environment may be marked as `N/A` with justification.

## 1. GPU Support

- At least one node with NVIDIA GPU installed.
- NVIDIA drivers installed and verified with `nvidia-smi`.
- CUDA and NVIDIA Container Toolkit installed.
- Kubernetes NVIDIA Device Plugin deployed.
- Validation: Create a test Pod that requests GPU and runs `nvidia-smi`.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**: Example justification if N/A: "No GPU nodes available in this environment."

## 2. Scheduling and AI Workload Management

- Kubernetes cluster supports advanced scheduling for AI workloads (affinity, taints/tolerations, resource requests).
- Optional: Volcano, Kueue, or other group scheduling solutions installed.
- Validation: Submit test workload with specific GPU and CPU requirements.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 3. Gateway API

- Cluster supports Gateway API for traffic management.
- Optional: Advanced routing, weighted traffic splitting, header-based routing.
- Validation: Deploy test inference service and configure Gateway API rules.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 4. AI/MLOps Platform

- AI/MLOps platform deployed on the cluster.
- Platform publicly accessible with documentation for installation, configuration, and examples.
- Validation: Platform operators can reproduce installation and run sample workloads.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 5. Monitoring and Metrics

- Prometheus/Grafana or equivalent monitoring deployed.
- GPU metrics exposed via DCGM exporter.
- AI workload metrics exposed (Ray, Kubeflow, or custom operators).
- Metrics follow Prometheus/OpenTelemetry standards.
- Validation: Prometheus targets include GPU nodes and AI workloads; metrics visible and queryable.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 6. Dynamic Resource Allocation (DRA)

- Cluster supports DRA API for fine-grained resource requests.
- Optional: Custom cluster provider or NodePool implementation for GPU nodes.
- Validation: Submit workload requesting specific accelerator resources.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 7. Horizontal/Vertical Pod Autoscaling

- HPA/VPA supports AI workloads with accelerator resources.
- Validation: Run workload and verify scaling based on custom metrics.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 8. AI Operators

- At least one CRD-based AI operator deployed (e.g., Ray, Kubeflow).
- Operator Pods and Webhooks running correctly.
- Validation: Create custom resource and verify coordination.
- **Status**: ⬜ Implemented / ⬜ N/A
- **Notes**:

## 9. Conformance Submission

- Submit:
  - YAML manifests
  - Screenshots of deployed workloads
  - Logs demonstrating platform compliance
  - Documentation of installation, configuration, and validation
- **Status**: ⬜ Completed / ⬜ N/A
- **Notes**:

------

> ⚠️ **Notes**: Items that cannot be implemented due to hardware or environment limitations may be marked as `N/A` with a clear justification.

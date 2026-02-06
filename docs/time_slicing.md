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

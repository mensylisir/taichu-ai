# Gateway API Support

This document describes the prerequisites, installation steps, and validation required for Gateway API support in a Kubernetes cluster for CNCF Kubernetes AI Conformance.

## Prerequisites

Create cluster with Kubernetes version >= 1.34.
Helm version 3 or above installed.

## Install Gateway API CRDs and Gateway Controller

```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.6.2 -n envoy-gateway-system --create-namespace
```

## To view the CRDs installed on your cluster, run the following command
```
(base) root@ai18:~/wode/gateway-helm# kubectl get crds | grep "gateway.networking.k8s.io"
backendtlspolicies.gateway.networking.k8s.io          2026-01-14T06:14:19Z
gatewayclasses.gateway.networking.k8s.io              2026-01-14T06:14:19Z
gateways.gateway.networking.k8s.io                    2026-01-14T06:14:19Z
grpcroutes.gateway.networking.k8s.io                  2026-01-14T06:14:19Z
httproutes.gateway.networking.k8s.io                  2026-01-14T06:14:19Z
referencegrants.gateway.networking.k8s.io             2026-01-14T06:14:20Z
tcproutes.gateway.networking.k8s.io                   2026-01-14T06:19:11Z
tlsroutes.gateway.networking.k8s.io                   2026-01-14T06:19:11Z
udproutes.gateway.networking.k8s.io                   2026-01-14T06:19:11Z
```

## Verify that the controller Pods are running:

```
(base) root@ai18:~/wode/gateway-helm# kubectl get pod -n envoy-gateway-system 
NAME                             READY   STATUS    RESTARTS      AGE
envoy-gateway-7869bbc889-rgxfb   1/1     Running   2 (15h ago)   41h
(base) root@ai18:~/wode/gateway-helm# 
```

## Gateway and HTTPRoute Example

1. Create a Gateway to handle AI inference traffic:

```
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: envoy-default
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: NodePort
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: envoy-default
    namespace: envoy-gateway-system
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eg
spec:
  gatewayClassName: eg
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

2. Create a ServiceAccount for the backend workload
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
```

3. Create a backend workload for testing
```
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
    service: backend
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      serviceAccountName: backend
      containers:
        - image: gcr.io/k8s-staging-gateway-api/echo-basic:v20231214-v1.0.0-140-gf544a46e
          imagePullPolicy: IfNotPresent
          name: backend
          ports:
            - containerPort: 3000
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

4. Create an HTTPRoute to route requests to your AI service:

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: backend
spec:
  parentRefs:
    - name: eg
  hostnames:
    - "example.test.rdev.tech"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: backend
          port: 3000
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

## Validation

1. Ensure Gateway and HTTPRoute resources are created:

```
(base) root@ai18:~/wode/gateway-example# kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
backend               1/1     1            1           8s
(base) root@ai18:~/wode/gateway-example# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
backend-77d4d5968-z9fpg               1/1     Running   0          14s
(base) root@ai18:~/wode/gateway-example# kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
backend      ClusterIP   10.68.249.212   <none>        3000/TCP   18s
(base) root@ai18:~/wode/gateway-example# kubectl get gateway
NAME   CLASS   ADDRESS       PROGRAMMED   AGE
eg     eg      172.30.1.18   True         21s
(base) root@ai18:~/wode/gateway-example# kubectl get httproute
NAME      HOSTNAMES                    AGE
backend   ["example.test.rdev.tech"]   28s
(base) root@ai18:~/wode/gateway-example# 
```

2. Send HTTP requests to the Gateway and verify routing:

```
(base) root@ai18:~/wode/gateway-example# curl --verbose --header "Host: example.test.rdev.tech" http://172.30.1.18:31389/
*   Trying 172.30.1.18:31389...
* Connected to 172.30.1.18 (172.30.1.18) port 31389 (#0)
> GET / HTTP/1.1
> Host: example.test.rdev.tech
> User-Agent: curl/7.87.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: application/json
< x-content-type-options: nosniff
< date: Fri, 16 Jan 2026 01:30:25 GMT
< content-length: 480
< 
{
 "path": "/",
 "host": "example.test.rdev.tech",
 "method": "GET",
 "proto": "HTTP/1.1",
 "headers": {
  "Accept": [
   "*/*"
  ],
  "User-Agent": [
   "curl/7.87.0"
  ],
  "X-Envoy-External-Address": [
   "172.20.121.128"
  ],
  "X-Forwarded-For": [
   "172.20.121.128"
  ],
  "X-Forwarded-Proto": [
   "http"
  ],
  "X-Request-Id": [
   "49a87027-b484-43be-a165-4d77e63904f4"
  ]
 },
 "namespace": "default",
 "ingress": "",
 "service": "",
 "pod": "backend-77d4d5968-z9fpg"
* Connection #0 to host 172.30.1.18 left intact
}(base) root@ai18:~/wode/gateway-example# 
```

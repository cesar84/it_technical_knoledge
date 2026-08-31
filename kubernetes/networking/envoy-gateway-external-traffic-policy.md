# Envoy Gateway loses connectivity on node drain

## Problem
Draining the worker node made the MetalLB external IP unreachable.

## Why
The Envoy Gateway Service used `externalTrafficPolicy: Local`. MetalLB only announces the IP from a node with a healthy Envoy pod, so traffic dropped when the pod moved.

## How to solve it
Create an `EnvoyProxy` resource with `externalTrafficPolicy: Cluster`, and reference it from the `GatewayClass` via `parametersRef`.

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        externalTrafficPolicy: Cluster
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: <your-gatewayclass-name>
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: envoy-gateway-system
```

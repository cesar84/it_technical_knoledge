# Service Port vs Container Port in NetworkPolicy

## The Problem

You define a Service with `port: 80`. The pod listens on `containerPort: 3000`. Your NetworkPolicy egress rule must allow port `3000`, not `80`. This doc explains why.

## Key Fact

A Service has no listening process. It is a virtual construct. kube-proxy implements it as iptables or IPVS rules on each node.

## How Port Translation Works

1. A client sends a packet to the Service's ClusterIP, on the Service `port`.
2. kube-proxy rewrites the packet. This is called DNAT (Destination Network Address Translation).
3. kube-proxy changes the destination IP to a pod IP. It changes the destination port to the pod's `targetPort`.
4. The packet arrives at the pod's network interface with the new destination port already set.

## Example

Service spec:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: monitoring-prometheus-grafana
spec:
  ports:
  - name: http-web
    port: 80
    protocol: TCP
    targetPort: grafana
```

Pod spec:

```yaml
containers:
- name: grafana
  ports:
  - containerPort: 3000
    name: grafana
    protocol: TCP
  - containerPort: 9094
    name: gossip-tcp
    protocol: TCP
  - containerPort: 6060
    name: profiling
    protocol: TCP
```

`targetPort: grafana` is a name, not a number. kube-proxy looks up this name in the pod's `ports[]` list. It finds `containerPort: 3000` because that entry has `name: grafana`. The other container ports (`9094`, `6060`) are ignored. The Service only maps the one port it defines.

## Where NetworkPolicy Fits

NetworkPolicy is enforced by the CNI (for example, Calico). The CNI checks traffic at the pod's network interface. By that point, DNAT already happened.

This means:

| Layer | Sees this port |
|---|---|
| Client / HTTPRoute `backendRefs.port` | Service port (80) |
| Service definition | Service port (80) → targetPort (grafana → 3000) |
| NetworkPolicy | Container port (3000) |
| Pod process | Container port (3000) |

## Rule of Thumb

- **HTTPRoute and Service specs**: use the Service `port`.
- **NetworkPolicy ingress/egress rules**: use the pod's actual `containerPort`.

If `targetPort` is a name, resolve it against the pod spec to find the number. If `targetPort` is already a number, or missing (in which case it defaults to `port`), use that value directly.

## How to Check

Find the Service's targetPort:

```bash
kubectl -n <namespace> get svc <service-name> -o yaml | grep -A 5 ports:
```

Find what a named targetPort resolves to:

```bash
kubectl -n <namespace> get pod -l <pod-selector> \
  -o jsonpath='{range .items[0].spec.containers[*]}{.name}{": "}{.ports}{"\n"}{end}'
```

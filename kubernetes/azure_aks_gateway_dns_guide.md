# Azure AKS Gateway API: Public DNS & Multiple FQDN Architecture Guide

## 1. Overview & Context

When you deploy apps on Azure Kubernetes Service (AKS) with the Kubernetes Gateway API (NGINX Gateway Fabric), you often want readable hostnames instead of raw public IPs.

Azure gives every Public IP resource a built-in DNS label. You get this before you buy or configure a custom domain. The label has this format:

```
<custom-label>.<azure-region>.cloudapp.azure.com
```

Example: `scapp.westeurope.cloudapp.azure.com`

## 2. Core Constraint: The Azure Public IP 1:1 Limitation

Azure enforces one strict rule: one Azure Public IP address can have at most **one** native Azure DNS label (`*.cloudapp.azure.com`).

This rule leaves you three architectural approaches.

### Approach A: 1 Public IP / 1 Gateway + Path-based Routing

- FQDN: `scapp.westeurope.cloudapp.azure.com`
- Traffic splits by URL path:
  - `/app1` -> App 1 Backend
  - `/app2` -> App 2 Backend

### Approach B: Multiple Gateways / Multiple Public IPs (Pure Azure FQDNs)

- Public IP 1: `scapp1.westeurope.cloudapp.azure.com` -> Gateway 1 -> App 1
- Public IP 2: `scapp2.westeurope.cloudapp.azure.com` -> Gateway 2 -> App 2
- This needs multiple Gateways, GatewayClasses, and NginxGateway configs.

### Approach C: 1 Gateway / 1 Public IP + Multiple CNAMEs (Custom Domain or DuckDNS)

Multiple distinct hostnames (for example `app1.domain.com` and `app2.domain.com`) point to the same single Gateway through DNS CNAME records.

## 3. How the Azure DNS Label Annotation Works

When Kubernetes creates a Service of type `LoadBalancer`, the Azure cloud controller manager provisions an Azure Public IP in the cluster node resource group (`MC_*`).

You set this annotation on the Service to configure the DNS label on the Public IP:

```
service.beta.kubernetes.io/azure-dns-label-name: "scapp"
```

## 4. NGINX Gateway Fabric Integration Specifics

With NGINX Gateway Fabric, annotations placed directly on a `Gateway` resource are **not** forwarded to the auto-generated Kubernetes Service.

Instead, NGINX Gateway Fabric uses custom parameters:

1. `NginxGateway` (CRD): holds the Service annotations.
2. `GatewayClass`: references the `NginxGateway` CR through `spec.parametersRef`.
3. `Gateway`: references the `GatewayClass` through `spec.gatewayClassName`.

A `GatewayClass` points to only one `NginxGateway` configuration. So distinct Azure labels on distinct Gateways need a dedicated chain for each:

```
NginxGateway (scapp1) -> GatewayClass (nginx-app1) -> Gateway 1 (Public IP 1)
NginxGateway (scapp2) -> GatewayClass (nginx-app2) -> Gateway 2 (Public IP 2)
```

## 5. Complete Dual Gateway Manifests (2 Public IPs, 2 Azure FQDNs)

### 5.1 NginxGateway Configurations

```yaml
apiVersion: gateway.nginx.org/v1alpha1
kind: NginxGateway
metadata:
  name: nginx-config-app1
  namespace: app
spec:
  infrastructure:
    service:
      annotations:
        service.beta.kubernetes.io/azure-dns-label-name: "scapp1"
---
apiVersion: gateway.nginx.org/v1alpha1
kind: NginxGateway
metadata:
  name: nginx-config-app2
  namespace: app
spec:
  infrastructure:
    service:
      annotations:
        service.beta.kubernetes.io/azure-dns-label-name: "scapp2"
```

### 5.2 GatewayClasses

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-app1
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  parametersRef:
    group: gateway.nginx.org
    kind: NginxGateway
    name: nginx-config-app1
    namespace: app
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-app2
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  parametersRef:
    group: gateway.nginx.org
    kind: NginxGateway
    name: nginx-config-app2
    namespace: app
```

### 5.3 Gateways

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-app1
  namespace: app
  labels:
    app.kubernetes.io/managed-by: Helm
    helm.toolkit.fluxcd.io/name: common-app
    helm.toolkit.fluxcd.io/namespace: app
spec:
  gatewayClassName: nginx-app1
  listeners:
    - name: http-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-app2
  namespace: app
  labels:
    app.kubernetes.io/managed-by: Helm
    helm.toolkit.fluxcd.io/name: common-app
    helm.toolkit.fluxcd.io/namespace: app
spec:
  gatewayClassName: nginx-app2
  listeners:
    - name: http-listener
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Same
```

### 5.4 HTTPRoutes

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-app1
  namespace: app
spec:
  parentRefs:
    - name: gateway-app1
  hostnames:
    - "scapp1.westeurope.cloudapp.azure.com"
  rules:
    - backendRefs:
        - name: app1-service
          port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-app2
  namespace: app
spec:
  parentRefs:
    - name: gateway-app2
  hostnames:
    - "scapp2.westeurope.cloudapp.azure.com"
  rules:
    - backendRefs:
        - name: app2-service
          port: 80
```

## 6. Operational Alternative: Direct Service Annotation

If you don't want to create custom `NginxGateway` CRDs and want to test immediately on an already running Gateway:

1. Identify the generated Service:
   ```bash
   kubectl get svc -n app -l gateway.networking.k8s.io/gateway-name=gateway-app
   ```
2. Annotate the Service directly:
   ```bash
   kubectl annotate svc <SERVICE_NAME> -n app \
     service.beta.kubernetes.io/azure-dns-label-name=scapp1 \
     --overwrite
   ```
3. Check the Public IP in Azure:
   ```bash
   az network public-ip list \
     --query "[?dnsSettings.domainNameLabel=='scapp1'].{IP:ipAddress, FQDN:dnsSettings.fqdn}" \
     --output table
   ```

> Note: Direct annotations applied by hand can be reverted if Flux CD or Helm triggers a full reconciliation cycle.

## 7. Alternative: 1 Gateway with Multiple Hostnames (CNAME / DuckDNS)

If you want multiple distinct hostnames without paying for extra Public IPs or maintaining duplicate Gateway objects:

1. Create CNAME records that point to your single Azure FQDN:
   ```
   app1.yourdomain.com -> CNAME -> scapp1.westeurope.cloudapp.azure.com
   app2.yourdomain.com -> CNAME -> scapp1.westeurope.cloudapp.azure.com
   ```
   (Or use DuckDNS subdomains that point to the Public IP address.)

2. Route both on the same Gateway with host-based matching:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-domain-app1
  namespace: app
spec:
  parentRefs:
    - name: gateway-app
  hostnames:
    - "app1.yourdomain.com"
  rules:
    - backendRefs:
        - name: app1-service
          port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: route-domain-app2
  namespace: app
spec:
  parentRefs:
    - name: gateway-app
  hostnames:
    - "app2.yourdomain.com"
  rules:
    - backendRefs:
        - name: app2-service
          port: 80
```

## 8. Comparison Table

| Feature | Multi-Gateway (Azure FQDN) | Single Gateway + CNAMEs | Single Gateway (Paths) |
|---|---|---|---|
| Public IPs Needed | 2 | 1 | 1 |
| Azure Cost | 2x Public IP charges | 1x Public IP | 1x Public IP |
| Domain Cost | Free | Free (DDNS) or Registrar | Free |
| Gateway / Classes | 2 of each | 1 of each | 1 of each |
| URL Format | scapp1... / scapp2... | app1.com / app2.com | scapp.../app1 & /app2 |
| Traffic Isolation | Separate IPs & data planes | Shared IP, Host header | Shared IP, path prefix |

## Official References

- [Microsoft Learn: Use a static IP and DNS label with AKS Load Balancer](https://learn.microsoft.com/en-us/azure/aks/static-ip)
- [F5 NGINX Gateway Fabric: NginxGateway Custom Resource](https://docs.nginx.com/nginx-gateway-fabric/overview/crds/nginx-gateway/)
- [F5 NGINX Gateway Fabric: GatewayClass Configuration](https://docs.nginx.com/nginx-gateway-fabric/how-to/traffic-management/gatewayclass/)
- [Kubernetes Gateway API: HTTPRoute Specification](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.HTTPRoute)
# Reusing Values Across HelmReleases in Flux CD

When managing multiple `HelmRelease` resources in a GitOps workflow with Flux CD, sharing configuration values—such as domain names, global tags, cluster environments, or resource allocations—prevents configuration drift and reduces boilerplate.

Below is an overview of the primary approaches, prioritized with **Flux Post-Build Variable Substitution** first for its readability and simplicity.

---

## 1. Primary Pattern: Flux Post-Build Variable Substitution (Recommended for Readability)

Flux's Kustomize controller (`kustomize.toolkit.fluxcd.io`) supports a `postBuild` phase that can substitute environment-like variables into arbitrary manifest fields before they are applied to the cluster.

This behaves like standard templating/variable interpolation, making your manifests clean, readable, and decoupled from environment-specific values.

### A. Inline Substitution in Flux `Kustomization`

Define common key-value pairs directly in the Flux `Kustomization` manifest:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform-apps
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: apps
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: ./apps
  prune: true
  postBuild:
    substitute:
      CLUSTER_DOMAIN: "prod.example.com"
      ENVIRONMENT: "production"
      LOG_LEVEL: "info"
      REPLICA_COUNT: "3"
      ENABLE_METRICS: "true"
```

### B. Consuming Variables in Multiple `HelmRelease` Manifests

Inside the directory targeted by the Flux Kustomization (`./apps`), all manifests can reuse those variables:

**`apps/api-service.yaml`:**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: api-service
  namespace: apps
spec:
  chart:
    spec:
      chart: ./charts/microservice
      sourceRef:
        kind: GitRepository
        name: fleet-infra
  values:
    replicaCount: ${REPLICA_COUNT}             # Parsed as integer: 3
    ingress:
      host: "api.${CLUSTER_DOMAIN}"            # String concatenation
    env:
      ENVIRONMENT: "${ENVIRONMENT}"
      LOG_LEVEL: "${LOG_LEVEL:=info}"          # Default fallback syntax
    metrics:
      enabled: ${ENABLE_METRICS}               # Parsed as boolean: true
```

**`apps/worker-service.yaml`:**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: worker-service
  namespace: apps
spec:
  chart:
    spec:
      chart: ./charts/worker
      sourceRef:
        kind: GitRepository
        name: fleet-infra
  values:
    replicaCount: ${REPLICA_COUNT}
    logging:
      level: "${LOG_LEVEL:=info}"
    broker:
      endpoint: "queue.${CLUSTER_DOMAIN}"
```

### C. Advanced: Loading Variables from Cluster `ConfigMap` / `Secret`

Instead of hardcoding values inside your Git repo's Flux `Kustomization`, you can read them dynamically from a pre-existing cluster `ConfigMap` or `Secret` using `substituteFrom`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform-apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-settings
        optional: false
      - kind: Secret
        name: cluster-secrets
        optional: true
```

### D. Crucial Syntax & Type Rules

1. **Type Coercion (Quotes matter):**
   - `${REPLICA_COUNT}` without quotes parses as a native YAML number (`3`).
   - `"${REPLICA_COUNT}"` with quotes forces a string (`"3"`).
   - `${ENABLE_METRICS}` without quotes parses as a YAML boolean (`true`).
2. **Default Fallbacks:**
   - Use bash-style parameter expansion: `${LOG_LEVEL:=info}`. If `LOG_LEVEL` is unset, it resolves to `"info"`.
3. **Escaping Variables:**
   - If your chart or template uses `${VAR}` syntax intended for runtime evaluation (e.g., container entrypoints), escape it with a caret: `^${CONTAINER_VAR}`.

---

## 2. Alternative Pattern: Shared `ConfigMap` / `Secret` via `spec.valuesFrom`

If you need to share entire blocks of values (like global resource limits, registry credentials, or tracing configurations), Helm Controller provides native support via `valuesFrom`.

### A. Shared ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: global-helm-values
  namespace: apps
data:
  values.yaml: |
    global:
      domain: internal.example.com
      imageRegistry: cr.example.com/platform
      imagePullSecrets:
        - name: registry-creds
    resources:
      limits:
        memory: 512Mi
        cpu: 500m
      requests:
        memory: 256Mi
        cpu: 250m
```

### B. Consuming via `valuesFrom`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: payment-service
  namespace: apps
spec:
  chart:
    spec:
      chart: ./charts/service
      sourceRef:
        kind: GitRepository
        name: fleet-infra
  valuesFrom:
    - kind: ConfigMap
      name: global-helm-values
      valuesKey: values.yaml
  values:
    # Service-specific overrides merge over valuesFrom
    service:
      port: 8080
    resources:
      limits:
        memory: 1Gi   # Overrides 512Mi from global-helm-values
```

### Key Considerations:
- **Merging Behavior:** `spec.values` always overrides keys defined in `valuesFrom`. If multiple items exist in `valuesFrom`, later entries take precedence.
- **Config Tracking:** Set `targetPath` or `valuesKey` to pull specific subsections if desired.

---

## 3. Alternative Pattern: Native Kustomize Patches

If you want pure native Kustomize without Flux post-build substitutions or extra ConfigMaps, use Kustomize JSON 6902 or Strategic Merge patches targeting `HelmRelease` objects.

```yaml
# apps/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service-a.yaml
  - service-b.yaml

patches:
  - target:
      kind: HelmRelease
    patch: |-
      - op: add
        path: /spec/values/global
        value:
          domain: example.com
          environment: staging
```

---

## Summary Comparison Matrix

| Method | Best For | Readability | Maintenance Overhead |
| :--- | :--- | :--- | :--- |
| **1. Flux Post-Build Substitution** | Environment-wide constants, strings, ints, ports, domains | **Highest** (inline `${VAR}` syntax) | Low; configured at the Kustomization root |
| **2. `spec.valuesFrom`** | Large, structured YAML blocks (global limits, registries) | **Moderate** (manifests need explicit `valuesFrom` pointer) | Low to medium; requires managing a shared ConfigMap |
| **3. Kustomize Patches** | Pure Kustomize pipelines without runtime/Flux controller dependencies | **Lowest** (JSONPatch paths can be brittle) | Medium; patch targets must match resource schema |

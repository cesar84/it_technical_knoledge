# External Secrets Operator (ESO) Integration with AKS & Azure Key Vault

This guide explains the architectural purpose, security rationale, and operational mechanics behind each step of integrating the **External Secrets Operator (ESO)** with **Azure Kubernetes Service (AKS)** and **Azure Key Vault (AKV)** using **Azure AD Workload Identity**.

---

## Architectural Overview

Traditional secret management in Kubernetes often requires storing static credentials (like Azure Service Principal secrets) inside cluster secrets. This architecture uses **Azure AD Workload Identity**, eliminating long-lived credentials entirely:

```
+-----------------------------------------------------------------------------------+
| AKS Cluster                                                                       |
|                                                                                   |
|  [Namespace: app]                                                                 |
|   +-------------------+      OIDC Token Exchange       +------------------------+ |
|   | ServiceAccount    | -----------------------------> | Microsoft Entra ID     | |
|   | (eso-sa)          |                                | (Managed Identity)     | |
|   +---------+---------+                                +-----------+------------+ |
|             |                                                      |              |
|             v                                                      | RBAC Role    |
|   +-------------------+        Fetches Secrets                     v              |
|   | SecretStore       | <----------------------------+  +-----------------------+ |
|   | (azure-kv)        |                              |  | Azure Key Vault       | |
|   +---------+---------+                              +--| (project02kvdev)      | |
|             |                                           +-----------------------+ |
|             v                                                                     |
|   +-------------------+    Creates/Updates                                        |
|   | ExternalSecret    | -------------------------> +-----------------------+      |
|   | (db-secret)       |                            | Kubernetes Secret     |      |
|   +-------------------+                            | (db-secret)           |      |
|                                                    +-----------------------+      |
+-----------------------------------------------------------------------------------+
```

---

## Breakdown of Steps and Their Purpose

### Environment Configuration (`Env setting`)

```bash
RG=project-02-dev-rg
IDENTITY=id-eso
KV=project02kvdev
AKS=project-02-dev-aks
KV_ID=$(az keyvault show -n $KV -g $RG --query id -o tsv)
NS=app
SA=eso-sa
```

* **Purpose:** Sets parameterized variables to ensure deterministic, reproducible execution.
* **Why it matters:** 
  * Captures `KV_ID` early to scope Azure Role-Based Access Control (RBAC) specifically to this vault instance rather than subscription-wide.
  * Explicitly binds the target Kubernetes namespace (`app`) and ServiceAccount (`eso-sa`) where workloads and secret stores reside.

---

### Step 1: Enable Cluster Capabilities (`Cluster flags`)

```bash
az aks update \
  --resource-group $RG \
  --name $AKS \
  --enable-oidc-issuer \
  --enable-workload-identity
```

* **Purpose:** Enables OpenID Connect (OIDC) token issuance and the Workload Identity mutating webhook on AKS.
* **Why it matters:**
  * `--enable-oidc-issuer`: Configures AKS to act as an OIDC Identity Provider (IdP) by exposing a public discovery document (`/.well-known/openid-configuration`) and signing keys. This allows Microsoft Entra ID to validate JSON Web Tokens (JWTs) issued by the AKS API server.
  * `--enable-workload-identity`: Installs an admission controller in AKS that detects annotated ServiceAccounts and injects required environment variables (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_FEDERATED_TOKEN_FILE`) and projected token volumes into pods.

---

### Step 2: Create Managed Identity & Extract Identifiers

```bash
az identity create --name $IDENTITY --resource-group $RG

CLIENT_ID=$(az identity show -n $IDENTITY -g $RG --query clientId -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
PRINCIPAL_ID=$(az identity show -n $IDENTITY -g $RG --query principalId -o tsv)
OIDC=$(az aks show -n $AKS -g $RG --query oidcIssuerProfile.issuerUrl -o tsv)
```

* **Purpose:** Provisions a User-Assigned Managed Identity in Microsoft Entra ID and queries runtime identity metadata.
* **Why it matters:**
  * **User-Assigned Managed Identity (`IDENTITY`):** Serves as the cloud-side identity representing the Kubernetes ServiceAccount. It has no password or certificate to rotate.
  * **`CLIENT_ID` & `TENANT_ID`:** Required by the Kubernetes ServiceAccount annotation so the workload knows which Azure identity to assume.
  * **`PRINCIPAL_ID` (Object ID):** The security identifier used to assign RBAC roles to this identity.
  * **`OIDC`:** The unique OIDC issuer URL of the AKS cluster needed for federated credential configuration.

---

### Step 3: Grant RBAC Access to Key Vault (`Access to the vault`)

```bash
az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $KV_ID
```

* **Purpose:** Grants the Managed Identity authorization to read/manage secrets within the designated Azure Key Vault instance.
* **Why it matters:**
  * Uses Azure Key Vault RBAC permission model instead of legacy Vault Access Policies.
  * Scoping the role to `$KV_ID` follows the **Principle of Least Privilege (PoLP)**, ensuring the identity cannot access other Key Vaults in the same resource group or subscription.
  * *Note on Role Selection:* While `Key Vault Secrets Officer` grants full management (read, write, delete), `Key Vault Secrets User` is the recommended minimum role if ESO only needs read access to secrets.

---

### Step 4: Configure Federated Identity Credential (`Trust the service account`)

```bash
az identity federated-credential create \
  --name fic-eso \
  --identity-name $IDENTITY \
  --resource-group $RG \
  --issuer $OIDC \
  --subject system:serviceaccount:${NS}:${SA} \
  --audience api://AzureADTokenExchange
```

* **Purpose:** Establishes a bidirectional trust relationship between Microsoft Entra ID and the AKS ServiceAccount.
* **Why it matters:**
  * **`--issuer`:** Restricts trust strictly to tokens signed by this specific AKS cluster's OIDC issuer.
  * **`--subject` (`system:serviceaccount:${NS}:${SA}`):** Restricts identity assumption to a specific ServiceAccount (`eso-sa`) within a specific namespace (`app`). Any other pod or namespace cannot exchange tokens for this Azure identity.
  * **`--audience` (`api://AzureADTokenExchange`):** Ensures Entra ID only accepts tokens generated specifically for Azure AD token exchange requests.

---

### Step 5: Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

* **Purpose:** Deploys the ESO controller, webhook, and Custom Resource Definitions (CRDs) into the dedicated `external-secrets` namespace.
* **Verification Command:**
  ```bash
  test $(kubectl api-resources --api-group=external-secrets.io | grep -cv NAME) -eq 6 && echo "OK: ESO installed" || echo "ERRORO: ESO is not properly installed"
  ```
  * Checks that all 6 core CRDs (`SecretStore`, `ClusterSecretStore`, `ExternalSecret`, `ClusterExternalSecret`, `PushSecret`, `ACRRefreshToken`) are registered.

---

### Step 6: Deploy Custom Resources (`eso.yaml`)

#### 1. The `SecretStore` Resource

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-kv
  namespace: app
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: "https://project02kvdev.vault.azure.net"
      serviceAccountRef:
        name: eso-sa
```

* **Purpose:** Defines the authentication mechanism and endpoint configuration for the upstream secret store (Azure Key Vault).
* **Key Configuration Items:**
  * `namespace: app`: Scoped only to the `app` namespace.
  * `authType: WorkloadIdentity`: Directs ESO to use projected Azure AD federated tokens.
  * `vaultUrl`: Points directly to the target Azure Key Vault DNS name.
  * `serviceAccountRef.name: eso-sa`: Binds the store directly to the federated ServiceAccount configured in Step 4.

#### 2. The `ExternalSecret` Resource

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-secret
  namespace: app
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: SecretStore
    name: azure-kv
  target:
    name: db-secret          # the Kubernetes Secret ESO creates
    creationPolicy: Owner
  data:
    - secretKey: db-password    # key inside the Kubernetes Secret
      remoteRef:
        key: password     # name of the secret in Key Vault
```

* **Purpose:** Declares the synchronization mapping between Azure Key Vault secrets and native Kubernetes `Secret` resources.
* **Key Configuration Items:**
  * `refreshInterval: 1h`: Periodically polls Key Vault every hour to ensure secrets stay synchronized and rotations propagate automatically.
  * `secretStoreRef`: Refers back to the `azure-kv` `SecretStore` configured above.
  * `target.creationPolicy: Owner`: Ensures ESO manages the lifecycle of the resulting native Kubernetes `Secret`; if the `ExternalSecret` is deleted, the native secret is cleaned up.
  * `data[]`: Maps the Key Vault secret name (`key: password`) into the Kubernetes secret data map (`secretKey: db-password`).

---

## Security Best Practices & Summary

1. **Zero Static Credentials:** No client secrets, API tokens, or certificates exist inside Kubernetes manifests or git repositories.
2. **Namespace Isolation:** The `SecretStore` and federated credential are tied strictly to `system:serviceaccount:app:eso-sa`.
3. **Automatic Rotation:** Secrets updated in Azure Key Vault are refreshed in the Kubernetes cluster according to `refreshInterval`.
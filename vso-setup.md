# Vault Secrets Operator (VSO) — clear step-by-step install + manifests



> ⚠️ Security notes up front
>
> * Prefer a dedicated namespace (I use `vault` in examples).
> * Avoid `skipTLSVerify: true` in production — use `caCertSecretRef` with a real CA.
> * The Vault role you point to in `VaultAuth.kubernetes.role` must already exist in Vault and be configured to allow the service account(s) you use (you already created that role in your other .md — great).

---

## Quick variables (replace as needed)


# change these for your environment
NAMESPACE="vault"                       # namespace where VSO will run (or choose vault-secrets-operator)
RELEASE_NAME="vault-secrets-operator"   # helm release name
VAULT_ADDR="http://43.204.140.166:8200" # change to your Vault address (use https in prod)
VAULT_ROLE="k8s-role"                   # Vault role created in Vault (must exist already)
SA_IN_NAMESPACE="default"               # service account VSO will use / or create a specific SA
```

---

## 1) Install VSO via Helm


# add repo and update
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# install VSO into the chosen namespace (creates the namespace)
helm install --namespace "$NAMESPACE" --create-namespace "$RELEASE_NAME" hashicorp/vault-secrets-operator
```

Wait for the operator pods to be Ready:


kubectl get pods -n "$NAMESPACE" -w
# or:
kubectl get pods -n "$NAMESPACE"
```

If you have a different namespace convention, change `NAMESPACE` above (examples later assume `vault`).

---

## 2) Find the service account the operator is running as (recommended)

The ClusterRole/ClusterRoleBinding you apply must give the **operator's service account** permission to create service account tokens / TokenReview.

Get the SA used by each deployment in the namespace:


# show deployments and their serviceAccountName
kubectl get deploy -n "$NAMESPACE" -o=jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.template.spec.serviceAccountName}{"\n"}{end}'
```

Pick the SA name returned (example below uses `vault-secrets-operator-controller-manager` or `vault-secrets-operator` depending on chart version). If you want to create your own SA instead, choose a name (e.g. `vso-sa`) and update the cluster-role-binding subject and `VaultAuth` later accordingly.

---

## 3) Create ClusterRole + ClusterRoleBinding (token reviewer perms)

Create `vault-token-reviewer.yaml` (replace `<SA_NAME>` and `<NAMESPACE>` with the SA and namespace from step 2 — example uses `vault` namespace and `vault-secrets-operator-controller-manager` serviceaccount):

```yaml
# vault-token-reviewer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vault-token-reviewer
rules:
- apiGroups: [""]
  resources: ["serviceaccounts/token"]
  verbs: ["create"]
- apiGroups: ["authentication.k8s.io"]
  resources: ["tokenreviews"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-token-reviewer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vault-token-reviewer
subjects:
- kind: ServiceAccount
  name: <SA_NAME>        # e.g. vault-secrets-operator-controller-manager
  namespace: <NAMESPACE> # e.g. vault
```

Apply it:


kubectl apply -f vault-token-reviewer.yaml
```

(If you prefer a one-liner that doesn't require editing the YAML, you can create the binding directly:

`
kubectl create clusterrolebinding vault-token-reviewer-binding \
  --clusterrole=vault-token-reviewer \
  --serviceaccount="$NAMESPACE:$SA_NAME"


---

## 4) (Optional but recommended) Create a CA secret for Vault TLS (instead of skipTLSVerify)

If Vault uses TLS with a CA not trusted by cluster nodes, create a secret with `ca.crt`:


# on the machine where you have the CA file (base64-safe)
kubectl create secret generic vault-ca-cert -n "$NAMESPACE" --from-file=ca.crt=./ca.crt
````

You can then reference this secret from `VaultConnection` (see step 5). This is the safer option for prod.

---

## 5) Create VaultConnection (tells VSO how to reach Vault)

Create `vault-connection.yaml` (replace `address`, and if you used step 4, set `caCertSecretRef` and remove `skipTLSVerify: true`):

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: <NAMESPACE>
spec:
  # address to the Vault server
  address: "<VAULT_ADDR>"          # e.g. "https://vault.example.com:8200"
  skipTLSVerify: true              # DEV only; set to false in prod
  # caCertSecretRef: "vault-ca-cert"  # uncomment if you created vault-ca-cert in step 4
```

Apply:


kubectl apply -f vault-connection.yaml
```

Verify connection status (operator will validate the connection and report status):


kubectl get vaultconnections -n "$NAMESPACE"
kubectl describe vaultconnection vault-connection -n "$NAMESPACE"
```

If `status.valid` is `false` check `kubectl describe` events and operator logs.

---

## 6) Create VaultAuth (configure how VSO authenticates to Vault)

`VaultAuth` tells VSO which Vault role (in Vault) to use and which service account/token to present.

Create `vault-authentication.yaml` (edit `serviceAccount` to match the SA used by the VSO controller or the SA in the target namespace that should be used for authentication; `role` must match the Vault role you created in Vault):

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: <NAMESPACE>
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: <VAULT_ROLE>           # e.g. k8s-role (must already be created in Vault)
    serviceAccount: <SA_TO_USE>  # serviceAccount used to create tokens for auth; e.g. default or vso-sa
```

Apply:


kubectl apply -f vault-authentication.yaml
```

Verify:


kubectl get vaultauth -n "$NAMESPACE"
kubectl describe vaultauth vault-auth -n "$NAMESPACE"
```

---

## 7) Example: VaultStaticSecret (sync a Vault secret into Kubernetes)

Once `VaultConnection` + `VaultAuth` are valid, you can create a CR to sync a Vault secret into K8s. Example (KV v2):

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: demo-secret
  namespace: app-namespace
spec:
  vaultConnectionRef: vault-connection
  vaultAuthRef: vault-auth
  # for KV v2, include the path under data/..., or set the engine type in spec
  engine: kv-v2
  vaultPath: secret/data/demo            # adjust for your mount & path
  secret:
    name: demo-secret-k8s
    type: Opaque
    keys:
      - key: username
        property: data.username
      - key: password
        property: data.password
```

Apply in the **consumer namespace** (where you want the Kubernetes secret to live):


kubectl apply -f demo-vaultstaticsecret.yaml -n app-namespace
kubectl get secret demo-secret-k8s -n app-namespace
```

(See HashiCorp docs for VaultStaticSecret, VaultDynamicSecret, VaultPKISecret etc.)

---

## 8) Verify operator behavior & logs


# operator pods
kubectl get pods -n "$NAMESPACE"

# operator logs (pick a pod name)
kubectl logs -n "$NAMESPACE" deploy/vault-secrets-operator-controller-manager -c manager

# inspect VaultConnection / VaultAuth status events
kubectl describe vaultconnection vault-connection -n "$NAMESPACE"
kubectl describe vaultauth vault-auth -n "$NAMESPACE"
```

Common problems:

* `VaultConnection` invalid → check address, reachability from cluster, TLS CA.
* `VaultAuth` failing → check `issuer`/`kubernetes` config in Vault auth method and that the Vault role is correctly bound to the SA name/namespace.
* Token review / token create denied → confirm the ClusterRole/ClusterRoleBinding in step 3 targets the actual SA the operator is using.

---

## 9) Cleanup (delete VSO and RBAC created by you)

When you want to remove VSO and these custom resources:


# uninstall helm release
helm uninstall "$RELEASE_NAME" -n "$NAMESPACE"

# delete the CRs you created
kubectl delete -f vault-authentication.yaml -n "$NAMESPACE"
kubectl delete -f vault-connection.yaml -n "$NAMESPACE"

# delete clusterrole & binding you applied
kubectl delete clusterrole vault-token-reviewer
kubectl delete clusterrolebinding vault-token-reviewer-binding

# (optionally) delete the namespace (careful if other resources live there)
kubectl delete ns "$NAMESPACE"
```

---

## Helpful one-line checklist (order to run)

1. `helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update`
2. `helm install --namespace $NAMESPACE --create-namespace $RELEASE_NAME hashicorp/vault-secrets-operator`
3. Find operator SA: `kubectl get deploy -n $NAMESPACE -o=jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.template.spec.serviceAccountName}{"\n"}{end}'`
4. Update & apply RBAC `vault-token-reviewer.yaml` (subject = SA from step 3)
5. (optional) `kubectl create secret generic vault-ca-cert -n $NAMESPACE --from-file=ca.crt=./ca.crt`
6. Apply `vault-connection.yaml` (set `address`, use `caCertSecretRef` in prod)
7. Apply `vault-authentication.yaml` (set `serviceAccount` and `role`)
8. Apply your VaultStaticSecret / VaultDynamicSecret CRs to sync secrets

---


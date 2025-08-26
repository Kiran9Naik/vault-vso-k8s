
# Enable Kubernetes auth in Vault (step-by-step)

> ⚠️ Security note: don't use `tls_disable = 1` or `0.0.0.0` in production. The examples below assume you have `vault` CLI installed, and that you can reach the Kubernetes API from the machine where you run the Vault configuration commands.

---

## Assumptions / prerequisites

* You can run `kubectl` against the target cluster (local kubeconfig or in-cluster context).
* You can run `vault` commands on the Vault server (or a machine with VAULT\_ADDR set and admin token).
* There is a `vault` namespace in the cluster (we show how to create it if missing).
* You want a Kubernetes service account bound to a Vault role that allows pods in namespace `vault` to read secrets.

---

## 1) (Recommended) Create a dedicated ServiceAccount for token review in the cluster

Run on a machine that has `kubectl` access to the cluster:


# create namespace if it doesn't exist
kubectl create ns vault || true

# create a dedicated service account in the 'vault' namespace
kubectl create serviceaccount vault-auth -n vault || true

# grant that SA the cluster role required to review tokens
kubectl create clusterrolebinding vault-auth-token-reviewer \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault:vault-auth || true


This gives the `vault-auth` SA the token-review capability (recommended). If you already use another SA with token reviewer permissions, you can use that instead.

---

## 2) Generate (or fetch) the ServiceAccount token (cluster-hosted machine)

You can generate a token and store it in an environment variable. Replace `vault-auth` with the SA you created (or `default` if you intentionally want to use the default SA).


# variables (change if needed)
SA_NAME="vault-auth"
SA_NAMESPACE="vault"

# generate token and store into variable
export JWT_TOKEN="$(kubectl create token ${SA_NAME} -n ${SA_NAMESPACE})"

# quick check (inspect JWT payload)
echo "$JWT_TOKEN" | cut -d '.' -f2 | base64 --decode | jq .
# get issuer from token payload (used later for 'issuer' field)
export ISSUER="$(echo "$JWT_TOKEN" | cut -d '.' -f2 | base64 --decode | jq -r .iss)"
echo "issuer: $ISSUER"


> If `kubectl create token` is unavailable on older clusters, you can get the token from the service account secret. (Modern clusters: `kubectl create token` is easiest.)

---

## 3) Capture Kubernetes API server host & CA cert (cluster-hosted machine)


# KUBE_HOST: API server URL (include https:// prefix from kubeconfig)
export KUBE_HOST="$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')"
echo "kube host: $KUBE_HOST"

# write the cluster CA into ca.crt (current kube-context)
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
  | base64 --decode > ca.crt && echo "Wrote ca.crt"


`KUBE_HOST` should be something like `https://<api-server-host>:6443`. `ca.crt` will be used by Vault to validate the kube API.

---

## 4) On the Vault server: enable & configure the Kubernetes auth method

Set `VAULT_ADDR` and ensure you are logged in with a Vault admin token (or run these where Vault CLI is authenticated):


export VAULT_ADDR='http://<vault-server>:8200'   # or your Vault URL
vault status                                      # confirm Vault is up


Enable Kubernetes auth (if not already enabled):


vault auth enable kubernetes || true


Now configure the Kubernetes auth method using values gathered earlier. The `token_reviewer_jwt` must be the token for a service account that can call the Kubernetes TokenReview API (that's why we recommended `vault-auth` with `system:auth-delegator`).


# ensure JWT_TOKEN, KUBE_HOST and ca.crt are accessible on the Vault server.
# If you generated them on another machine, copy them to the Vault server securely.

# Example (run on Vault server, assuming JWT_TOKEN env and ca.crt are present):
export JWT_TOKEN="<paste-or-export-jwt-here>"
export KUBE_HOST="<paste-kube-host-here>"  # e.g. https://10.0.0.1:6443
export ISSUER="<paste-issuer-here>"

vault write auth/kubernetes/config \
  token_reviewer_jwt="$JWT_TOKEN" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert=@ca.crt \
  issuer="$ISSUER"


**Tip:** if you exported `ISSUER` from the JWT payload earlier, use that value. The `issuer` field must match the `iss` claim in the service account token.

---

## 5) Create a Vault policy that allows access to secrets

Example policy that grants read access to `kv/*`:


cat > myapp-policy.hcl <<'EOF'
path "kv/*" {
  capabilities = ["read"]
}
EOF

vault policy write myapp-policy myapp-policy.hcl


**Note:** If your KV secret engine is mounted as KV v2, your policy path will need to be `path "kv/data/*" { ... }` (and you will use the v2 read/write API paths). Adjust per your secrets mount.

---

## 6) Create a Kubernetes auth role in Vault

This maps K8s ServiceAccount(s) → Vault policies:


vault write auth/kubernetes/role/k8s-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=vault \
  alias_name_source=serviceaccount_uid \
  policies=myapp-policy \
  ttl=24h


What these fields mean:

* `bound_service_account_names` — which SA names are allowed (comma-separated). Here: `default`.
* `bound_service_account_namespaces` — which namespaces (here: `vault`).
* `policies` — Vault policies to attach to tokens issued for this role.
* `alias_name_source=serviceaccount_uid` — recommended to map by unique SA UID (avoids confusion when SAs reused).
* `ttl` — issued token TTL.

Adjust `bound_service_account_names` / `namespaces` to match your app's SA + namespace (for example, use `vault-auth` if you want to bind that SA).

---

## 7) Test logging in with a Kubernetes token (quick verification)

From a machine that can reach Vault (or on the Vault server):


# use the same JWT_TOKEN from step 2 (or the token mounted in a pod)
export JWT_TOKEN="<your-k8s-sa-token>"

# attempt login
vault write auth/kubernetes/login role=k8s-role jwt="$JWT_TOKEN"


Expected result: Vault returns a `client_token`. You can set it to test secret access:


export VAULT_TOKEN="<client_token_returned>"
vault token lookup   # verifies token is valid
# try reading a secret (example)
vault kv get kv/my-secret


---

## 8) Useful verification / debug commands


# show enabled auth methods
vault auth list

# view kubernetes auth config
vault read auth/kubernetes/config

# show role details
vault read auth/kubernetes/role/k8s-role

# view policy
vault policy read myapp-policy

# Vault server logs if the systemd unit is used:
sudo journalctl -u vault.service -f


Common issues:

* **`token review` unauthorized**: The token used as `token_reviewer_jwt` must come from an SA with token-review permission (`system:auth-delegator`). Recreate the token from an SA bound to that cluster role.
* **Issuer mismatch**: If Vault fails to validate the token, check the `iss` in the JWT (`echo "$JWT_TOKEN" | cut -d '.' -f2 | base64 --decode | jq .iss`) and set `issuer` to the exact value.
* **CA / host unreachable**: Ensure `kubernetes_host` is reachable from the Vault server and `kubernetes_ca_cert` matches the cluster CA.

---

## Example: minimal copy-paste (cluster machine → vault server)

1. On **cluster machine**:


kubectl create ns vault || true
kubectl create serviceaccount vault-auth -n vault || true
kubectl create clusterrolebinding vault-auth-token-reviewer \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault:vault-auth || true

export JWT_TOKEN="$(kubectl create token vault-auth -n vault)"
export ISSUER="$(echo "$JWT_TOKEN" | cut -d '.' -f2 | base64 --decode | jq -r .iss)"
export KUBE_HOST="$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')"
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' \
  | base64 --decode > ca.crt
# copy JWT_TOKEN and ca.crt to your Vault server securely


2. On **Vault server** (after copying `ca.crt` and setting `VAULT_ADDR` and admin auth):


vault auth enable kubernetes || true

vault write auth/kubernetes/config \
  token_reviewer_jwt="$JWT_TOKEN" \
  kubernetes_host="$KUBE_HOST" \
  kubernetes_ca_cert=@ca.crt \
  issuer="$ISSUER"

cat > myapp-policy.hcl <<'EOF'
path "kv/*" {
  capabilities = ["read"]
}
EOF
vault policy write myapp-policy myapp-policy.hcl

vault write auth/kubernetes/role/k8s-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=vault \
  alias_name_source=serviceaccount_uid \
  policies=myapp-policy \
  ttl=24h


---


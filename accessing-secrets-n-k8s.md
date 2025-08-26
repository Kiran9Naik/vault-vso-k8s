# VSO — final test & secret-injection walkthrough (step-by-step)


1. test Kubernetes auth against Vault (external test)
2. create a `VaultStaticSecret` to sync a Vault secret into Kubernetes
3. verify the secret in-cluster and decode it
4. run a sample Pod that *consumes* the secret (env + mounted file)
5. debug and cleanup steps

All commands assume you have `kubectl` access to the cluster (and the correct context), `vault` CLI available where indicated, and that the Vault Secrets Operator (VSO), `VaultConnection` and `VaultAuth` are already configured.

---

## 0) Quick variables — edit for your environment


# change as needed
NAMESPACE="vault"             # namespace where VSO and the secret live
VAULT_ROLE="k8s-role"         # Vault role you created in Vault
SA_NAME="default"             # service account to test with (or the one VSO uses)
STATIC_SECRET_NAME="podcheck" # K8s secret name VSO will create (destination.name)
VAULT_STATIC_CR="vault-static-secret"  # the VaultStaticSecret resource name


---

## 1) External test: try a Kubernetes login to Vault

Run this from any machine that can reach Vault (and has `vault` CLI and kubectl):


vault write auth/kubernetes/login \
  role=<role-name> \
  jwt="$(kubectl create token <service-account-name> -n <namespace-name>)"


Example (replace placeholders):


vault write auth/kubernetes/login \
  role=${VAULT_ROLE} \
  jwt="$(kubectl create token ${SA_NAME} -n ${NAMESPACE})"


### Expected success output (example) and what the main fields mean

You should get output like:


Key                                       Value
---                                       -----
token                                     hvs.... (client token)
token_accessor                            ... 
token_duration                            24h
token_renewable                           true
policies                                  ["default" "myapp-policy"]
token_meta_service_account_name           default
token_meta_service_account_namespace      vault
token_meta_role                           k8s-role


***token** — the Vault client token you can use to call Vault APIs.
***token\_duration** — how long this token is valid.
***policies** — Vault policies attached to this token.
***token\_meta\_**\* — metadata: which K8s service account / namespace / role were used.
  If you get this, your Vault `kubernetes` auth method is wired correctly for token validation.

If the login fails: check `issuer` mismatch, token-review permissions of the token-reviewer SA (needs `system:auth-delegator`), CA/host reachability, and Vault logs (commands below).

---

## 2) VaultStaticSecret (create the CR to sync a Vault secret into Kubernetes)

### A) Example for **KV v1** (matches your original)

Create `vault-secret-injection.yml`:

yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-static-secret
  namespace: vault
spec:
  vaultAuthRef: vault-auth
  mount: kv
  type: kv-v1              # KV version = 1
  path: testsec            # path in Vault (for kv v1, no "data/" prefix)
  refreshAfter: 10s
  destination:
    create: true
    name: podcheck         # Kubernetes Secret name that will be created


Apply it:


kubectl apply -f vault-secret-injection.yml


### B) Example for **KV v2** (if your KV mount is KV v2)

If Vault’s KV engine is v2, use `kv-v2` and the path should point to the logical path (no explicit `/data/` in the CR — operator handles v2), e.g.:

yaml
spec:
  mount: kv
  type: kv-v2
  path: testsec            # Vault path under kv (data/testsec in Vault internal API)


---

## 3) Wait & verify the VaultStaticSecret succeeded

Check the CR status and the created Kubernetes Secret:


# check VaultStaticSecret status
kubectl get vaultstaticsecret -n ${NAMESPACE}
kubectl describe vaultstaticsecret vault-static-secret -n ${NAMESPACE}

# list K8s secrets in the namespace
kubectl get secret -n ${NAMESPACE}

# inspect the secret metadata/keys
kubectl get secret ${STATIC_SECRET_NAME} -n ${NAMESPACE} -o yaml


> The VaultSecretsOperator commonly stores the secret payload under a key named `_raw` (single blob) or separate keys depending on how the CR was defined. Inspect the secret to see the keys.

### Decode secret values

List keys:


kubectl get secret ${STATIC_SECRET_NAME} -n ${NAMESPACE} -o jsonpath='{.data}' | jq


Decode a specific key (example for `_raw`):


kubectl get secret ${STATIC_SECRET_NAME} -n ${NAMESPACE} -o jsonpath='{.data._raw}' | base64 --decode


Or decode an arbitrary key `<KEY>`:


kubectl get secret ${STATIC_SECRET_NAME} -n ${NAMESPACE} -o jsonpath="{.data.<KEY>}" | base64 --decode


---

## 4) If you don’t see the secret / it’s empty — quick debug checklist

1. `kubectl describe vaultstaticsecret vault-static-secret -n ${NAMESPACE}` — check events and status.
2. Operator logs:

   
   # find deployment/pod name first:
   kubectl get deploy -n ${NAMESPACE}
   # then view logs (replace <DEPLOY> with the operator deployment name)
   kubectl logs -n ${NAMESPACE} deploy/<DEPLOY> -c manager --tail=200
   

   Typical deployment name: `vault-secrets-operator-controller-manager` or `vault-secrets-operator`.
3. Check VaultConnectivity:

   
   kubectl describe vaultconnection vault-connection -n ${NAMESPACE}
   kubectl describe vaultauth vault-auth -n ${NAMESPACE}
   
4. Confirm Vault role binding matches the SA & namespace used by VSO (`bound_service_account_names` and `bound_service_account_namespaces` in Vault role).
5. Vault server logs & config:

   
   vault status
   vault auth list
   vault read auth/kubernetes/config
   vault read auth/kubernetes/role/k8s-role
   sudo journalctl -u vault.service -n 200  # if you run Vault as systemd
   

---

## 5) Sample Pod manifests to consume the created K8s Secret

### A) Pod reading secret using `env` (secretKeyRef)

`pod-env.yaml`:

yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-test
  namespace: vault
spec:
  containers:
  - name: tester
    image: busybox
    command: ["sh", "-c", "echo SECRET (from env): $SECRET_RAW ; sleep 3600"]
    env:
      - name: SECRET_RAW
        valueFrom:
          secretKeyRef:
            name: podcheck   # must match destination.name in VaultStaticSecret
            key: _raw        # key inside the K8s secret (adjust if different)
  restartPolicy: Never


Apply & test:


kubectl apply -f pod-env.yaml
kubectl logs -n vault pod/secret-env-test
# or exec into pod
kubectl exec -n vault -it pod/secret-env-test -- sh -c 'echo $SECRET_RAW'


### B) Pod mounting secret as files (volume mount)

`pod-volume.yaml`:

yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-test
  namespace: vault
spec:
  containers:
  - name: tester
    image: busybox
    command: ["sh", "-c", "echo 'Content of /etc/secret/_raw:' ; cat /etc/secret/_raw || true ; sleep 3600"]
    volumeMounts:
      - name: secret-vol
        mountPath: /etc/secret
        readOnly: true
  restartPolicy: Never
  volumes:
    - name: secret-vol
      secret:
        secretName: podcheck   # the K8s secret created by VSO


Apply & test:


kubectl apply -f pod-volume.yaml
kubectl logs -n vault pod/secret-volume-test
# or:
kubectl exec -n vault -it pod/secret-volume-test -- sh -c 'cat /etc/secret/_raw'


> If the secret key is not `_raw` (check secret YAML), adjust `key:` in the env or mount path.

---

## 6) Cleanup commands (remove created resources)


# delete test pods
kubectl delete pod secret-env-test secret-volume-test -n ${NAMESPACE} --ignore-not-found

# remove the VaultStaticSecret CR (operator will delete the Kubernetes Secret if configured)
kubectl delete -f vault-secret-injection.yml -n ${NAMESPACE}

# delete the actual K8s Secret (if left behind)
kubectl delete secret ${STATIC_SECRET_NAME} -n ${NAMESPACE} --ignore-not-found

# optionally remove VaultAuth / VaultConnection CRs (if created earlier)
kubectl delete -f vault-authentication.yaml -n ${NAMESPACE} --ignore-not-found
kubectl delete -f vault-connection.yaml -n ${NAMESPACE} --ignore-not-found

# operator uninstall (if you want)
helm uninstall vault-secrets-operator -n ${NAMESPACE}


---

## 7) Final tips & gotchas

***KV v1 vs v2** — be explicit in your `VaultStaticSecret` `type` field: `kv-v1` or `kv-v2`. VSO handles the differences, but your `path` semantics change slightly.
***Permissions** — ensure the service account the operator uses has token-create/tokenreviews permission (ClusterRole/ClusterRoleBinding).
***TLS** — avoid `skipTLSVerify: true` in prod. Use `caCertSecretRef` and store CA cert as a k8s secret.
***Token TTL/policy** — if your Vault role has restrictive policies or TTLs, the created K8s Secret content and refresh behavior will reflect that.
***Refresh frequency** — `refreshAfter` controls how often the operator reconciles/re-reads the secret; pick a reasonable value for your use-case.
***Operator logs** are the fastest way to find why a secret did not sync.

---


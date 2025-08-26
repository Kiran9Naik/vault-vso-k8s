you can install either in default namespace/or in specific namespace (Suggestion: better to go for separate)
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault-secrets-operator hashicorp/vault-secrets-operator

once the installation done just create these files and apply in same order 

vi vault-token-reviewer.yaml       ============> for cluster role and rolebinding
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
  name: default
  namespace: <you're namespace>


$kubectl apply -f  vault-token-reviewer.yaml


vi vault-connection.yml     ==================> for establising connection to the VSO
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: vault
  name: vault-connection
spec:
  # address to the Vault server.
  address: http://43.204.140.166:8200
  skipTLSVerify: true

$kubectl apply -f  vault-connection.yaml


vi vault-authentication.yml         =============> to address the vault with k8s auth method throug the role 
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: vault
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: k8s-role
    serviceAccount: default      ---> be specific on this things 

$kubectl apply -f  vault-authentication.yaml




run commands in cluster-hosted machine:
---------------------------------------------------------------------------------------------------------------
kubectl create token default -n vault     -----> create $JWT_TOKEN
echo "$TOKEN" | cut -d '.' -f2 | base64 --decode | jq .     ---> to check whether there is any blockage like issuer
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}'  --------------> for $KUBE_HOST
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode > ca.crt && echo ca.crt  ----------------> for certificate of K8S

run the below commands on vault installed server:
---------------------------------------------------------------------------------------------------------------
$Vault auth enable kubernetes
$vault write auth/kubernetes/config token_reviewer_jwt="<paste generated token here/pass it as env value vall the value>" kubernetes_host="<k8s-hostname>" kubernetes_ca_cert=@ca.crt issuer="https://oidc.<endpoint>/id/<idnumber>"

Just create a policy to access the role of k8s into the secrets:

cat > myapp-policy.hcl <<EOF
path "kv/*" {
  capabilities = ["read"]
}
EOF
$vault policy write myapp-policy myapp-policy.hcl


vault write auth/kubernetes/role/k8s-role \
  bound_service_account_names=default \
  bound_service_account_namespaces=vault \
  alias_name_source=serviceaccount_uid \
  policies=myapp-policy \
  ttl=24h

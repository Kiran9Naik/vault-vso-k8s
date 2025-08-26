once the VSO setup is done clearly 

just if you want to test externally try this command

vault write auth/kubernetes/login \
  role=<role-name> \
  jwt="$(kubectl create token <service-account-name> -n <namespace-name>)"
if you get the response like this:
====================================================================
Key                                       Value
---                                       -----
token                                     hvs.CAESIOAKuvvHp0pTD5G3u0WODpfStOWP-CwJ718PAohOyLI5Gh4KHGh2cy51NHY3U0E1MkJrREw3STRHZHRrbFp1SjM
token_accessor                            utWAdMg1qEDDJYL4PaYB4M53
token_duration                            24h
token_renewable                           true
token_policies                            ["default" "myapp-policy"]
identity_policies                         []
policies                                  ["default" "myapp-policy"]
token_meta_service_account_secret_name    n/a
token_meta_service_account_uid            ee22686b-118e-413d-8ea4-d7402cd2e54e
token_meta_role                           k8s-role
token_meta_service_account_name           default
token_meta_service_account_namespace      vault
====================================================================== 
just reference for you 

then proceed with secret injection test 

vault-secret-injection.yml          --------------> for static secret creation in k8s 

apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-static-secret
  namespace: vault
spec:
  vaultAuthRef: vault-auth
  mount: kv
  type: kv-v1               # <-- Use kv here for KV version 1
  path: testsec        # No /data suffix
  refreshAfter: 10s
  destination:
    create: true
    name: podcheck


$kubectl apply -f vault-secret-injection.yml


once it is done check the secret data :
=========================================
kubectl get secret -n <namespace> <secret-name> -o jsonpath='{.data._raw}' | base64 --decode

you will be able to get the response if not check the VSO pod logs 
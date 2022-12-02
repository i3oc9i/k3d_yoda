# Vault Deployment Guide

## 1. Configure hashicorp repo

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

## 2. Cretae vault namespace
```
kubectl create ns vault
```

## 3. Use Consul as Vault backend
```
helm search repo hashicorp/consul --versions

helm install consul hashicorp/consul \
  --namespace vault \
  --version 1.0.1 \
  -f ./values/consul-values.yaml 
```

## 4. Create the TLS secret (autosigned)
```
cfssl gencert -initca ./tls/ca-csr.json | cfssljson -bare ./crt/ca

kubectl -n vault create secret tls tls-ca \
 --cert ./crt/ca.pem  \
 --key ./crt/ca-key.pem

cfssl gencert \
  -ca=./crt/ca.pem \
  -ca-key=./crt/ca-key.pem \
  -config=./tls/ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ./tls/ca-csr.json | cfssljson -bare ./crt/vault

kubectl -n vault create secret tls tls-server \
  --cert ./crt/vault.pem \
  --key ./crt/vault-key.pem
```

## 5. Deploy Vault
```
helm search repo hashicorp/vault --versions

helm install vault hashicorp/vault \
  --namespace vault \
  --version 0.22.1 \
  -f ./values/vault-values.yaml
```

## 6. Unsealing Vault
```
kubectl -n vault exec vault-0 -- vault operator init -format=json > ./secrets/vault.json

for vault in vault-0 vault-1 vault-2
do
  echo ">>> unsealing ${vault} ..."
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[0]' ./secrets/vault.json)
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[1]' ./secrets/vault.json)
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[2]' ./secrets/vault.json)
done
```

## 7. Check Vault Status
```
for vault in vault-0 vault-1 vault-2
do
  kubectl -n vault exec ${vault} -- vault status
done
```
## 8. Acess Web UI (Optional)
```
kubectl -n vault port-forward svc/vault-ui 8200
```
Access the web UI and API at https://127.0.0.1:8200/

## 9. Enable Kubernetes Authentication
```
kubectl -n vault exec vault-0 -- vault login $(jq -r '.root_token' ./secrets/vault.json)

kubectl -n vault exec vault-0 -- vault auth enable kubernetes

kubectl -n vault exec vault-0 -- sh -c 'vault write auth/kubernetes/config \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer=https://kubernetes.default.svc.cluster.local'
```

## 10. Basic Secret Injection (Exemple)

### 10.1 Create a role for the `example-app`
Cretae a `basic-secret-role` in vault mapping the kubernetes service account `basic-secret` to a `basic-secret-policy`
for applications deployed into the `example-app` namespace  
```
kubectl -n vault exec vault-0 -- vault write auth/kubernetes/role/basic-secret-role \
   bound_service_account_names=basic-secret \
   bound_service_account_namespaces=example-app \
   policies=basic-secret-policy \
   ttl=1h
```

Create the policy `basic-secret-policy` in vault allowing the service account `basic-secret` to read secrets
```
kubectl -n vault exec vault-0 -- sh -c '
cat << EOF | vault policy write basic-secret-policy -
path "secret/basic-secret/*" {
  capabilities = ["read"]
}
EOF'
```

Enable key value secrets in vault for `secrets` folder 
```
kubectl -n vault exec vault-0 -- vault secrets enable -path=secret/ kv
```

store `secret-user` and `secret-db` secrets in `basic-secret` vault folder
```
kubectl -n vault exec vault-0 -- vault kv put secret/basic-secret/secret-user  username=onavi password=QwErT-AsDfg-12345-%%
kubectl -n vault exec vault-0 -- vault kv put secret/basic-secret/secret-db    username=admin password=YuIoP-ZxCvB-67890-%%
```

### 10.2 Deploy the `example-app` and verify secrets injection
```
kubectl apply -f ./example-app/deployment.yaml
```

Once the pod is ready, verify the secrets injected 
```
POD=$(kubectl -n example-app get pod -o json | jq -r '.items[0].metadata.name')
kubectl -n example-app exec ${POD} -- cat /vault/secrets/user
kubectl -n example-app exec ${POD} -- cat /vault/secrets/db
```

## 11. Using External Secrets Operator (Exemple)




# Vault Deployment Guide

## Configure hashicorp repo
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

## Use Consul as Vault backend
```
helm search repo hashicorp/consul --versions

kubectl create ns vault

helm install consul hashicorp/consul \
  --namespace vault \
  --version 1.0.1 \
  -f ./values/consul-values.yaml 
```

## Create the TLS secret 
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

## Deploy Vault
```
helm search repo hashicorp/vault --versions

helm install vault hashicorp/vault \
  --namespace vault \
  --version 0.22.1 \
  -f ./values/vault-values.yaml
```

## Initialising Vault
```
kubectl -n vault exec -it vault-0 -- vault operator init -format=json > ./secrets/vault.json

for vault in vault-0 vault-1 vault-2
do
  echo ">>> unsealing ${vault} ..."
  kubectl -n vault exec -it ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[0]' ./secrets/vault.json)
  kubectl -n vault exec -it ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[1]' ./secrets/vault.json)
  kubectl -n vault exec -it ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[2]' ./secrets/vault.json)
done 

for vault in vault-0 vault-1 vault-2
do
  kubectl -n vault exec -it ${vault} -- vault status
done

```
## Web UI
```
kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui :8200
```
Access the web UI [here]("https://localhost/")

## Enable Kubernetes Authentication
```
kubectl -n vault exec -it vault-0 -- vault login $(jq -r '.root_token' ./secrets/vault.json)

kubectl -n vault exec -it vault-0 -- vault auth enable kubernetes

kubectl -n vault exec -it vault-0 -- sh -c 'vault write auth/kubernetes/config \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer=https://kubernetes.default.svc.cluster.local'
```


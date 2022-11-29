# Vault Deployment Guide

## Configure hashicorp repo
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

## Use Consul as Vault backend
```
helm search repo hashicorp/consul --versions

helm template consul hashicorp/consul \
  --namespace vault \
  --version 1.0.1 \
  -f ./values/consul-values.yaml \
  > ./manifests/consul.yaml

kubectl create ns vault
kubectl -n vault apply -f ./manifests/consul.yaml
```

## Create the TLS secret 
```
cfssl gencert -initca ./tls/ca-csr.json | cfssljson -bare /tmp/ca

kubectl -n vault create secret tls tls-ca \
 --cert /tmp/ca.pem  \
 --key /tmp/ca-key.pem

cfssl gencert \
  -ca=/tmp/ca.pem \
  -ca-key=/tmp/ca-key.pem \
  -config=./tls/ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ./tls/ca-csr.json | cfssljson -bare /tmp/vault

kubectl -n vault create secret tls tls-server \
  --cert /tmp/vault.pem \
  --key /tmp/vault-key.pem
```

## Deploy Vault
```
helm search repo hashicorp/vault --versions

helm template vault hashicorp/vault \
  --namespace vault \
  --version 0.22.1 \
  -f ./values/vault-values.yaml \
  > ./manifests/vault.yaml

kubectl -n vault apply -f ./manifests/vault.yaml
```

## Initialising Vault
```
kubectl -n vault exec -it vault-0 -- sh
kubectl -n vault exec -it vault-1 -- sh
kubectl -n vault exec -it vault-2 -- sh

vault operator init
vault operator unseal

kubectl -n vault exec -it vault-0 -- vault status
kubectl -n vault exec -it vault-1 -- vault status
kubectl -n vault exec -it vault-2 -- vault status

```
## Web UI
```
kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui :8200
```
Access the web UI [here]("https://localhost/")

## Enable Kubernetes Authentication
```
kubectl -n vault exec -it vault-0 -- sh 

vault login
vault auth enable kubernetes

vault write auth/kubernetes/config \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
issuer="https://kubernetes.default.svc.cluster.local"
exit
```


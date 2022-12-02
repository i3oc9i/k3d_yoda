# Vault Deployment Example

## 1. Configure hashicorp repo

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

## 2. Create vault namespace and local secret folder
```
kubectl create ns vault

mkdir -p ./.secrets
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
cfssl gencert -initca ./tls/ca-csr.json | cfssljson -bare ./.secrets/ca

kubectl -n vault create secret tls tls-ca \
 --cert ./.secrets/ca.pem  \
 --key ./.secrets/ca-key.pem

cfssl gencert \
  -ca=./.secrets/ca.pem \
  -ca-key=./.secrets/ca-key.pem \
  -config=./tls/ca-config.json \
  -hostname="vault,vault.vault.svc.cluster.local,vault.vault.svc,localhost,127.0.0.1" \
  -profile=default \
  ./tls/ca-csr.json | cfssljson -bare ./.secrets/vault

kubectl -n vault create secret tls tls-server \
  --cert ./.secrets/vault.pem \
  --key ./.secrets/vault-key.pem
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
kubectl -n vault exec vault-0 -- vault operator init -format=json > ./.secrets/vault-unseal-keys.json

for vault in vault-0 vault-1 vault-2
do
  echo ">>> unsealing ${vault} ..."
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[0]' ./.secrets/vault-unseal-keys.json)
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[1]' ./.secrets/vault-unseal-keys.json)
  kubectl -n vault exec ${vault} -- vault operator unseal $(jq -r '.unseal_keys_hex[2]' ./.secrets/vault-unseal-keys.json)
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
kubectl -n vault exec vault-0 -- vault login $(jq -r '.root_token' ./.secrets/vault-unseal-keys.json)

kubectl -n vault exec vault-0 -- vault auth enable kubernetes

kubectl -n vault exec vault-0 -- sh -c 'vault write auth/kubernetes/config \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token \
  kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer=https://kubernetes.default.svc.cluster.local'
```

## 10. Basic Secret Injection (Example)

### 10.1 Create a role and a policy for the `example-app`

Create a `basic-secret-role` in vault mapping the kubernetes service account `basic-secret` to a `basic-secret-policy`
for applications deployed into the `example-app` namespace  
```
kubectl -n vault exec vault-0 -- vault write auth/kubernetes/role/basic-secret-role \
   bound_service_account_names=basic-secret \
   bound_service_account_namespaces=example-app \
   policies=basic-secret-policy \
   ttl=87600h
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

## 10.2 Store a bunch of basic secrets

Enable key value secrets in vault for `secret/basic-secret` folder 
```
kubectl -n vault exec vault-0 -- vault secrets enable -path=secret/basic-secret/ kv
```

store `user` and `db` secrets in `basic-secret` vault folder
```
kubectl -n vault exec vault-0 -- vault kv put secret/basic-secret/user  username=onavi password=QwErT-AsDfg-12345-%%
kubectl -n vault exec vault-0 -- vault kv put secret/basic-secret/db    username=admin password=YuIoP-ZxCvB-67890-%%
```

### 10.3 Deploy the `example-app` and verify secrets injection
```
kubectl apply -f ./example-app/deployment.yaml
```

Once the pod is ready, verify the secrets injected 
```
POD=$(kubectl -n example-app get pod -o json | jq -r '.items[0].metadata.name')
kubectl -n example-app exec ${POD} -- cat /vault/secrets/user
kubectl -n example-app exec ${POD} -- cat /vault/secrets/db
```

## 11. Using External Secrets Operator (EOS) (Example)

## 11.1. Configure ESO repo
```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
```

## 11.2 Deploy ESO
```
helm search repo external-secrets/external-secrets --versions

helm install external-secrets \
    external-secrets/external-secrets \
    --namespace vault \
    --version 0.6.1 \
    --set installCRDs=true
```

## 11.3 Create a role and a policy for the external secrets

Create a `external-secrets-role` in vault mapping the kubernetes service account `external-secrets` to a `external-secrets-policy`
for `EOS` deployed into the `vault` namespace  
```
kubectl -n vault exec vault-0 -- vault write auth/kubernetes/role/external-secrets-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=vault \
  policies=external-secrets-policy \
  ttl=87600h
```

Create the policy `external-secrets-policy` in vault allowing the service account `external-secrets` to read secrets
```
kubectl -n vault exec vault-0 -- sh -c '
  cat << EOF | vault policy write external-secrets-policy -
  path "secret/external-secret/*" {
    capabilities = ["read", "list"]
  }
EOF'
```

## 11.4 Store a bunch of external secrets

Enable key value secrets in vault for `secret/external-secret` folder
```
kubectl -n vault exec vault-0 -- vault secrets enable -version=2 -path=secret/external-secret/ kv
```

store `user` and `db` secrets in `external-secret` vault folder
```
kubectl -n vault exec vault-0 -- vault kv put secret/external-secret/user  username=onavi password=QwErT-AsDfg-12345-%%-v2
kubectl -n vault exec vault-0 -- vault kv put secret/external-secret/db    username=admin password=YuIoP-ZxCvB-67890-%%-v2
```
## 11.5 Creating a Cluster Secret Store

Setting up a `ClusterSecretStore` holding the information for contacting the Vault secret provider
```
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-external-secret-backend
spec:
  provider:
    vault:
      server: "https://vault.vault.svc:8200"
      caProvider:
        type: "Secret"
        namespace: "vault"
        name: "tls-ca"
        key: "tls.crt"
      path: "secret/external-secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets-role"
EOF
```

## 11.6 Pulling secrets from the Secret Store
```
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  namespace: example-app
  name: pull-user-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-external-secret-backend
    kind: ClusterSecretStore
  target:
    name: user-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: user
        property: username
    - secretKey: password
      remoteRef:
        key: user
        property: password
EOF
```

## 11.7 Verify Pulled secrets
```
kubectl -n example-app get externalsecrets.external-secrets.io

kubectl -n example-app describe secrets user-secret
```

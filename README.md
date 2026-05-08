# vault-secrets-operator-setup
A quick setup of VSO using Minikube.

This guide assumes the following:
* Minikube up and running
* A 3 node Vault cluster with integrated storage backend

## Steps
### 1. Configure event notifications

**Exec into the leader pod:** 
```bash
kubectl exec -it vault-0 -- /bin/sh
```

**Enable a kv-v2 engine and add a secret:**
```bash
vault secrets enable -path=secret kv-v2

k exec $LEADER_POD -- vault kv put secret/repro-secret unique_id="$(date +%s)
```

**Enable k8s auth method:**
```bash
vault auth enable kubernetes
```

### 2. Create Event policy
```hcl
path "sys/events/subscribe/kv-v2*" {
  capabilities = ["read"]
}

path "secret/data/*" {
  capabilities = ["read", "list", "subscribe"]
  subscribe_event_types = ["*"]
}
```

**Copy the policy to the current leader pod and write the policy to Vault:**

```bash
k cp vso-policy.hcl $LEADER_POD:vault
vault policy write vso-policy vso-policy.hcl
```

### 3. Setup the K8's auth method

**Configure the Kubernetes auth method:**

```bash
vault write auth/kubernetes/config \
kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
```

**Configure the Kubernetes role:**


```bash
vault write auth/kubernetes/role/vso-role bound_service_account_names=vso-auth-sa \
bound_service_account_namespaces=$POD_NAMESPACE$ \
policies=vso-policy \
ttl=24h
```

### 4. Install the VSO helm chart
```bash
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
--namespace vault-secrets-operator-system \
--create-namespace
```

### 5. Configure the VSO CRD, create the VSO service account, and apply the manifest
```yaml
---
# 1. Define how to connect to Vault
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: default
spec:
  address: http://vault.default.svc.cluster.local:8200
---
# 2. Define how to authenticate (ServiceAccount must exist)
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: default
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: vso-role
    serviceAccount: vso-auth-sa
  vaultConnectionRef: vault-connection
---
# 3. Define the secret to sync and enable events
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: repro-static-secret
  namespace: default
spec:
  vaultAuthRef: vault-auth
  mount: secret
  type: kv-v2
  path: repro-secret
  destination:
    name: k8s-repro-secret
    create: true
```

**Create the service account:**
```bash
k create serviceaccount vso-auth-sa
```

**Apply the manifest:**
```bash
k apply -f vso-resources.yaml
```

### 6. Testing

**Read the static secret to check sync status and get the current secretMAC:**

```bash
kubectl get vaultstaticsecret repro-static-secret -o yaml

secretMAC: 9BxsHVFLe5oKGp8lKRd9DqZpRqUoHwYWv5jzKPtTtGc=
```

**Update the secret:**

```bash
k exec vault-0 -- vault kv put secret/repro-secret unique_id="$(date +%s)
```

**Read the static secret again to check the secretMAC has updated successfully:**

```bash
kubectl get vaultstaticsecret repro-static-secret -o yaml

status:
  conditions:
  - lastTransitionTime: "2026-05-05T13:54:25Z"
    message: Secret synced, horizon=52.619538554s
    observedGeneration: 3
    reason: Synced
    status: "True"
    type: SecretSynced
  - lastTransitionTime: "2026-05-05T13:54:25Z"
    message: VaultStaticSecretHealthy
    observedGeneration: 3
    reason: Healthy
    status: "True"
    type: Healthy
  - lastTransitionTime: "2026-05-05T13:54:25Z"
    message: VaultStaticSecretReady
    observedGeneration: 3
    reason: Ready
    status: "True"
    type: Ready
  lastGeneration: 3
  secretMAC: CSC05+8dQBh+88LtQyBx0+zZb0VtcFlhDoPVd5hsYV0=
```

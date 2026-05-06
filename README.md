# vault-secrets-operator-setup
A quick setup of VSO using Minikube.

This guide assumes the following:
* Minikube up and running
* A 3 node Vault cluster with integrated storage backend

## Steps
### configure event notifications

**exec into the leader pod** 
kubectl exec -it vault-0 -- /bin/sh

**enable a kv-v2 engine**
vault secrets enable -path=secret kv-v2

**enable k8s auth method**
vault auth enable kubernetes

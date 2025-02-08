# Vault-EMA Proof of Concept
This repository contains a Proof of Concept (PoC) for deploying HashiCorp Vault using Helm in a Kubernetes environment. The Vault instance is configured with High Availability (HA) using the Raft protocol. This setup is designed to run in the `ema-dev` namespace.
## Overview
The PoC demonstrates:
- Installing HashiCorp Vault using Helm
- Configuring Vault for HA with Raft
- Initializing and unsealing Vault
- Joining additional Vault pods to the Raft cluster
- reference url: `https://developer.hashicorp.com/vault/docs/platform/k8s/helm/examples/ha-with-raft`
**URL Endpoint:** `http://vault-ema-0.vault-ema-internal:8200`
**HINT:**  `http:\\pod-name.vault-svc-name:8200-port`
## Prerequisites
- Kubernetes cluster
- Helm installed and configured
- `kubectl` configured to interact with your cluster
## Installation Steps
### Step 1: Install Vault using Helm
Deploy Vault with HA and Raft support in the `ema-dev` namespace:
```bash
helm install vault-ema hashicorp/vault -n ema-dev \
  --set='server.ha.enabled=true' \
  --set='server.ha.raft.enabled=true'
```
### Step 2: Check Pod Status
Ensure all Vault pods are running:
```bash
kubectl get pods -n ema-dev
```
Output should initially show:
```
vault-ema-0                                 0/1     Running            0                2m51s
vault-ema-1                                 0/1     Running            0                2m38s
vault-ema-2                                 0/1     Running            0                2m38s
vault-ema-agent-injector-575b659775-sqlzs   1/1     Running            0                4m45s
```
### Step 3: Initialize Vault
Initialize Vault on the first pod:
```bash
kubectl exec -n ema-dev -ti vault-ema-0 -- vault operator init
```
Save the Unseal Keys and Initial Root Token for later use.
### Step 4: Unseal Vault
Unseal the `vault-ema-0` pod using the first three unseal keys:
```bash
kubectl exec -ti vault-ema-0 -n ema-dev -- vault operator unseal
```
Repeat this command three times, using a different key each time, until the output indicates the threshold has been met.
### Step 5: Join Additional Pods to Raft Cluster
#### Pod 1
Join `vault-ema-1` to the Raft cluster:
```bash
kubectl exec -ti vault-ema-1 -n ema-dev -- vault operator raft join http://vault-ema-0.vault-ema-internal:8200
```
Unseal `vault-ema-1`:
```bash
kubectl exec -ti vault-ema-1 -n ema-dev -- vault operator unseal
```
Repeat the unseal command three times with different keys.
#### Pod 2
Join `vault-ema-2` to the Raft cluster:
```bash
kubectl exec -ti vault-ema-2 -n ema-dev -- vault operator raft join http://vault-ema-0.vault-ema-internal:8200
```
Unseal `vault-ema-2`:
```bash
kubectl exec -ti vault-ema-2 -n ema-dev -- vault operator unseal
```
Repeat the unseal command three times with different keys.
### Step 6: Verify Pod Status
Verify all Vault pods are fully running:
```bash
kubectl get pods -n ema-dev
```
Expected output:
```
vault-ema-0                                 1/1     Running            0                30m
vault-ema-1                                 1/1     Running            0                30m
vault-ema-2                                 1/1     Running            0                30m
vault-ema-agent-injector-575b659775-sqlzs   1/1     Running            0                32m
```
## Conclusion
The HashiCorp Vault PoC with HA using Raft is now successfully deployed and ready for use. Make sure to securely store your unseal keys and root token. For further configuration and usage, refer to the HashiCorp Vault documentation.
```bash
vault-ema.ema-dev.svc.cluster.local  
 
Root Token: Tj7vYp82sgm
```
```bash
root@ET-Master-Node-2:~/emadev/sravan-vault-ema# kubectl exec -n ema-dev -ti vault-ema-0 -- vault operator init
Unseal Key 1: DedWqm6ZlhDT35WIXUII
Unseal Key 2: gqpoEkKrTwQ9Xf1nUt9
Unseal Key 3: 64FTh63mzswW29icJli
Unseal Key 4: Pg9OKswobzCv075Xm
Unseal Key 5: EOs7JmYi1Lspx
Initial Root Token: ugnJIWQATj7vYp82sgm
```
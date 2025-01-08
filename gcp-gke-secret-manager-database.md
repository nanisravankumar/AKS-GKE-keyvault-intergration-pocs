Here’s the updated **README.md** file with the additional references:

---

# GKE Secret Manager Integration for Database Secrets

This document provides a step-by-step guide for setting up a Proof of Concept (PoC) that integrates Google Kubernetes Engine (GKE) with GCP Secret Manager. In this PoC, database secrets are stored in GCP Secret Manager, and the GKE cluster is configured to access those secrets securely using Workload Identity and the `external-secrets` Kubernetes operator.

### Prerequisites:
- **Google Cloud Platform (GCP)** account with required permissions.
- **GKE Cluster** set up with Workload Identity and Secret Manager enabled.
- **kubectl** CLI tool installed and configured to interact with the GKE cluster.
- **Helm** installed for managing Kubernetes charts.
  
---

### Step 1: Create Secret in GCP Secret Manager

1. Navigate to **Secret Manager** in the GCP console.
2. Create a new secret named `db-config` and store the following secret value:

```json
{
  "dname": "postgres",
  "dbport": 5432,
  "dbpassword": "1225khbbvhjpassword",
  "dbipaddress": "20.54.25.45"
}
```

3. **Secret Name**: `db-config`
4. **Secret Value**:
   - `dname`: postgres
   - `dbport`: 5432
   - `dbpassword`: 1225khbbvhjpassword
   - `dbipaddress`: 20.54.25.45

---

### Step 2: Create GKE Cluster

Create a GKE cluster with the following configuration:

1. **Cluster Name**: `cluster-1`
2. **Cluster Location**: `us-central1`
3. **Enable Workload Identity**:
   - This enables GKE workloads to access Google Cloud APIs securely using Kubernetes service accounts.
4. **Enable Secret Manager** for the cluster.
5. **Enable GCE Instance Metadata** on NodePool.

To create the GKE cluster, run the following commands:

```bash
gcloud container clusters create cluster-1 \
  --location us-central1 \
  --enable-workload-identity \
  --metadata enable-oslogin=TRUE \
  --enable-secret-manager
```

---

### Step 3: Setup Workload Identity and Service Account in GKE

1. **Create Namespace** for your application:

```bash
kubectl create namespace control-tower
```

2. **Create a Kubernetes Service Account**:

```bash
kubectl create serviceaccount ct-service-account --namespace=control-tower
```

3. **Create a GCP Service Account**:

```bash
gcloud iam service-accounts create control-tower-gcpc --project=winged-amp-447209-i5
```

4. **Grant Secret Manager Access to the Service Account**:

```bash
gcloud projects add-iam-policy-binding winged-amp-447209-i5 \
  --member "serviceAccount:control-tower-gcpc@winged-amp-447209-i5.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"
```

5. **Grant Workload Identity User Role to the Kubernetes Service Account**:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  control-tower-gcpc@winged-amp-447209-i5.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:winged-amp-447209-i5.svc.id.goog[control-tower/ct-service-account]" \
  --project winged-amp-447209-i5
```

6. **Annotate the Kubernetes Service Account**:

```bash
kubectl annotate serviceaccount ct-service-account \
  --namespace control-tower \
  iam.gke.io/gcp-service-account=control-tower-gcpc@winged-amp-447209-i5.iam.gserviceaccount.com
```

---

### Step 4: Install External Secrets Operator

1. **Add the Helm repository for External Secrets**:

```bash
helm repo add external-secrets https://charts.external-secrets.io
```

2. **Install the External Secrets operator**:

```bash
helm install external-secrets external-secrets/external-secrets \
  -n control-tower \
  --create-namespace \
  --set installCRDs=true
```

---

### Step 5: Create SecretStore YAML File

1. **Create a SecretStore YAML file** (`secret-store.yaml`) to configure the connection to GCP Secret Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: sampleapp-store
  namespace: control-tower
spec:
  provider:
    gcpsm:
      projectID: winged-amp-447209-i5
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: cluster-1
          serviceAccountRef:
            name: ct-service-account
```

2. **Apply the SecretStore YAML**:

```bash
kubectl apply -f secret-store.yaml
```

---

### Step 6: Create ExternalSecret YAML File

Create an `external-secret.yaml` file to specify how to map secrets from GCP Secret Manager into Kubernetes Secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ext-secret
  namespace: control-tower
spec:
  refreshInterval: 1s
  secretStoreRef:
    kind: SecretStore
    name: sampleapp-store
  target:
    name: db-config-secret
    creationPolicy: Owner
  data:
    - secretKey: dname
      remoteRef:
        key: db-config
        property: dname
    - secretKey: dbport
      remoteRef:
        key: db-config
        property: dbport
    - secretKey: dbpassword
      remoteRef:
        key: db-config
        property: dbpassword
    - secretKey: dbipaddress
      remoteRef:
        key: db-config
        property: dbipaddress
```

Apply the `ExternalSecret`:

```bash
kubectl apply -f external-secret.yaml
```

---

### Step 7: Create Deployment YAML File

The following `deployment.yaml` file will create a deployment using the secrets as environment variables in the container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: control-tower
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: nginx:1.21.6
        ports:
        - containerPort: 80
        env:
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: db-config-secret
              key: dname
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: db-config-secret
              key: dbport
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-config-secret
              key: dbpassword
        - name: DB_IP_ADDRESS
          valueFrom:
            secretKeyRef:
              name: db-config-secret
              key: dbipaddress
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: control-tower
  labels:
    app: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the deployment:

```bash
kubectl apply -f deployment.yaml
```

---

### Step 8: Verify the Deployment and Service

1. **Check the Deployment status**:

```bash
kubectl get deployments -n control-tower
```

2. **Check the Service status**:

```bash
kubectl get svc my-app-service -n control-tower
```

---

### Conclusion

This PoC demonstrates how to securely store and access database secrets in GCP Secret Manager and use them within a GKE cluster using the `external-secrets` operator. By enabling Workload Identity and Secret Manager, you can ensure secure access to secrets within Kubernetes workloads.

---

### References

1. [Video Tutorial: Integrating GKE with Secret Manager](https://youtu.be/Wir-d6_h1dM?si=GOmjy_1EpyMEMKMc)
2. [Medium Article on Accessing Secret Manager Automatically in GKE](https://medium.com/@vinayak4451/access-secret-manager-automatically-to-gke-9d890c847c61)
3. [GKE Workload Identity Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

Let me know if you need further assistance!

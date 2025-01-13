```markdown
# Azure Key Vault Integration with AKS using Secrets Store CSI Driver

In this PoC, we will establish a secure connection between an Azure Key Vault and an AKS cluster to access secrets like `dbname`, `dbport`, `dbipaddress`, and `dbpassword`. 

## **Description**

The AKS cluster cannot directly fetch data from the Key Vault. To facilitate access, we enable the CSI driver and configure a Secret Provider Class. This configuration defines Key Vault details, and a managed identity is used to authenticate the connection. 

We will:

1. Create an AKS cluster and Key Vault.
2. Add secrets to the Key Vault.
3. Install and configure the Secrets Store CSI Driver.
4. Define a Secret Provider Class.
5. Deploy a workload to access the secrets.

## **Prerequisites**

- Azure CLI, `kubectl`, and Helm installed.
- Permissions to create Azure resources (AKS, Key Vault, etc.).

---

## **Steps**

### **1. Create Resources**

#### Create an AKS Cluster with public because for connectivity of API server - key vault will be private 

#### Create an Azure Key Vault from portal enable private endpoint for select both should be in same vnet aks and key vault
![image](https://github.com/user-attachments/assets/b4a98ab3-f936-4e5a-85e1-01bac03d6acf)


#### Add Secrets to the Key Vault
```bash
az keyvault secret set --vault-name azurekeyvaultestingaks --name dbname --value "dbnamevalue"
az keyvault secret set --vault-name azurekeyvaultestingaks --name dbport --value "5432"
az keyvault secret set --vault-name azurekeyvaultestingaks --name dbipaddress --value "192.168.1.1"
az keyvault secret set --vault-name azurekeyvaultestingaks --name dbpassword --value "securepassword"

```
![image](https://github.com/user-attachments/assets/8bac3007-164d-4b73-ab0a-b22fd88c6897)
---

### **2. Install the Secrets Store CSI Driver**

#### Add Helm Repository
```bash
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
```

#### Deploy the Driver
```bash
helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure
```

#### Verify Pods Are Running
```bash
kubectl get pods -n kube-system | grep secrets-store
```
![image](https://github.com/user-attachments/assets/d929023d-4288-4b09-bc6b-b74835d7b6f4)
---
### **3. Configure AKS for Key Vault Integration**

#### Enable Managed Identity for Key Vault Access  

1. Navigate to the Azure portal and go to the resource group **`MC_test-rg_akstestcluster_eastus`**.  
2. Locate the **`azurekeyvaultsecretsprovider-akstestcluster`** managed identity.  
   - This managed identity is created automatically when enabling the Secrets Store CSI Driver in the AKS cluster.  
3. Select **Azure Role Assignments** for the managed identity.  
4. Add a role assignment with the following details:  
   - **Role**: **Key Vault Secrets Officer**  
   - **Scope**: Key Vault  
   - **Key Vault**: Select **`azurekeyvaultestingaks`**.  
![image](https://github.com/user-attachments/assets/29c203e9-7e42-42c4-add7-89fc753bee30)

---

### **4. Create the Secret Provider Class**

#### YAML Configuration
Save the file below as `secretproviderclass.yaml`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
  namespace: ct-qa
spec:
  provider: azure
  parameters:
    usePodIdentity: "false" # Using managed identity
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "<userAssignedIdentityID>" # Replace with managed identity Client ID
    keyvaultName: azurekeyvaultestingaks
    objects: |
      array:
        - |
          objectName: dbport
          objectType: secret
          objectVersion: ""
        - |
          objectName: dbname
          objectType: secret
          objectVersion: ""
        - |
          objectName: dbipaddress
          objectType: secret
          objectVersion: ""
        - |
          objectName: dbpassword
          objectType: secret
          objectVersion: ""
    tenantID: "<tenantID>" # Replace with the Directory ID from Azure Portal
```

**Notes**:
- **`tenantID`**: Go to the Azure Key Vault > Overview > Copy the Directory ID.
![image](https://github.com/user-attachments/assets/c7f3d1f0-d99a-4a9c-a2b9-1f833745f322)


- **`userAssignedIdentityID`**: Navigate to `MC_test-rg_akstestcluster_eastus`, select the managed identity of `azurekeyvaultsecretsprovider-akstestcluster`, search in the Azure portal, and copy the `Client ID`.
![image](https://github.com/user-attachments/assets/5f3658d5-5fe8-4cd7-af0f-d375603fa97b)



Apply the configuration:
```bash
kubectl apply -f secretproviderclass.yaml
```

---

### **5. Deploy the Workload**

#### YAML Configuration
Save the file below as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-secrets-store-inline
  namespace: ct-qa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-secrets-store
  template:
    metadata:
      labels:
        app: busybox-secrets-store
    spec:
      containers:
      - name: busybox
        image: registry.k8s.io/e2e-test-images/busybox:1.29-4
        command:
          - "/bin/sleep"
          - "10000"
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-kvname"
```

**Note**: The `secretProviderClass` must match the name defined in `secretproviderclass.yaml`.
```bash
kubectl get secretProviderClass -n ct-qa
```
![image](https://github.com/user-attachments/assets/42f1b247-0c1f-4210-88b5-7884663e9e3e)



Deploy the workload:
```bash
kubectl apply -f deployment.yaml
```

Verify the pod:
```bash
kubectl get pods -n ct-qa

# it will list all secrets
kubectl exec busybox-secrets-store-inline-67cffbcd59-f4xbn -n ct-qa -- ls /mnt/secrets-store 

# it will display secret value
kubectl exec busybox-secrets-store-inline-67cffbcd59-f4xbn -n ct-qa -- cat /mnt/secrets-store/dbname

```

---

### **6. Go to portal and Update Secrets**

To update a secret (e.g., `dbname`), use:
```bash
az keyvault secret set --vault-name azurekeyvaultestingaks --name dbname --value "devdb"
```

Restart the deployment:
```bash
kubectl rollout restart deployment busybox-secrets-store-inline -n ct-qa

kubectl exec busybox-secrets-store-inline-67cffbcd59-f4xbn -n ct-qa -- cat /mnt/secrets-store/dbname
```
![image](https://github.com/user-attachments/assets/8639e90e-890b-40ba-84da-b4db268c526a)
---

## **References**

- [YouTube Walkthrough](https://youtu.be/8l9LRcUw3pA?si=aO7e1jqjTcQH6T3w)
- [Azure Docs - Service Principal Walkthrough](https://azure.github.io/secrets-store-csi-driver-provider-azure/docs/demos/standard-walkthrough/)

# Kubernetes Cluster Recovery and Migration Plan

## **Issue Summary**
Your AKS cluster is running **Kubernetes v1.19.11**, which is beyond its support lifecycle. The cluster is experiencing the following issues:
- **Service Principal (SP) and Cluster Certificates Expired**
- **Metrics-Server Pod CrashLoopBackOff** due to missing `/tmp` volume support (only available in v1.21+)
- **SecurityContext `readOnlyRootFilesystem:true` preventing self-signed certificate creation**
- **Cluster provisioning state manually set to `Succeeded` in backend, but core issues remain**

## **Recommended Actions**
### **Step 1: Attempt Temporary Fixes**
If you want to try recovering the cluster before migrating, follow these steps:

### **1.1 Stop the Cluster**
```sh
az aks stop --name myAKSCluster --resource-group myResourceGroup
```

### **1.2 Rotate Certificates**
```sh
az aks rotate-certs --name myAKSCluster --resource-group myResourceGroup
```

### **1.3 Update Service Principal Credentials**
1. Go to **Azure Active Directory** → **App Registrations**.
2. Find the **Service Principal (SP)** used by AKS.
3. Add a **new client secret**.
4. Update the AKS cluster with the new credentials:
   ```sh
   az aks update-credentials --name myAKSCluster --resource-group myResourceGroup --service-principal <appId> --client-secret <newSecret>
   ```

### **1.4 Attempt Metrics-Server Fix (Temporary Workaround)**
If `metrics-server` is failing due to a missing `/tmp` directory, modify the deployment:

```yaml
initContainers:
  - name: volume-mount-init
    image: busybox
    command: [ "sh", "-c", "mkdir -p /tmp && chmod 777 /tmp" ]
    volumeMounts:
      - mountPath: /tmp
        name: temp-volume
volumes:
  - name: temp-volume
    emptyDir: {}
```

⚠ **Warning:** This is a temporary workaround and not a sustainable fix.

---

## **Step 2: Migrate Workloads to a New Cluster**
Since your cluster is outdated and facing multiple issues, migration is the **best long-term solution**.

### **2.1 Create a New AKS Cluster**
```sh
az aks create --resource-group newResourceGroup --name newAKSCluster --kubernetes-version 1.27.3 --node-count 3 --enable-managed-identity
```

### **2.2 Backup and Export Existing Workloads**
1. **Export all Kubernetes resources**:
   ```sh
   kubectl get all -o yaml > backup.yaml
   ```
2. **Export secrets and config maps separately**:
   ```sh
   kubectl get secrets -o yaml > secrets.yaml
   kubectl get configmaps -o yaml > configmaps.yaml
   ```
3. **Take snapshots of persistent volumes** (if applicable).

### **2.3 Deploy Workloads to the New Cluster**
1. **Configure `kubectl` to use the new cluster:**
   ```sh
   az aks get-credentials --name newAKSCluster --resource-group newResourceGroup
   ```
2. **Apply the exported manifests**:
   ```sh
   kubectl apply -f backup.yaml
   kubectl apply -f secrets.yaml
   kubectl apply -f configmaps.yaml
   ```
3. **Update Ingress and DNS settings** if necessary.

### **2.4 Verify Deployment**
Check if all pods are running:
```sh
kubectl get pods -A
```
Ensure services are correctly exposed:
```sh
kubectl get svc -A
```

### **2.5 Decommission the Old Cluster (Once Migration is Successful)**
```sh
az aks delete --name myAKSCluster --resource-group myResourceGroup --yes --no-wait
```

## **Final Recommendation**
Since Kubernetes **v1.19.11 is unsupported**, fixing the cluster will only be a short-term solution. The best approach is to **migrate workloads to a new AKS cluster** running a **supported version (1.27+)**.

Would you like assistance automating this migration process?


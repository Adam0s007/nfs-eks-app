
# Large Scale Computing - LAB 07

## Components and their Roles

| # | Component | Role |
|---|-----------|------|
| 1 | **Helm release `nfs`** | Installs the NFS‑Ganesha server and external provisioner that dynamically creates NFS-backed PersistentVolumes. |
| 2 | **StorageClass `nfs-sc`** | Defines dynamic RWX storage provided by the NFS provisioner; referenced by PVCs. |
| 3 | **PVC `web-pvc`** | Requests 5 GiB shared storage (`ReadWriteMany`) from `nfs-sc`; mounted by the Job and nginx pod. |
| 4 | **Job `upload-content-job`** | Runs once, writes `index.html` into the PVC so that nginx has content to serve. |
| 5 | **Deployment `web-server`** | Runs an *nginx:alpine* pod that mounts the PVC and serves the HTML page. |
| 6 | **Service `web-server-svc`** | Type *LoadBalancer*; allocates an ELB and routes external HTTP traffic (port 80) to the `web-server` pod. |
| 7 | **Dynamic PV (created by provisioner)** | Backing NFS export that physically stores the data requested by `web-pvc`. |
 
Everything works under the **namespace `nfs`**
---
## Data‑flow between components

This section describes the flow of operations between components, following the numbered arrows shown on the architecture diagram:  
![View Architecture Diagram](./architecture-LSC-LAB07.svg)

| # | From → To | What happens on this step |
|---|-----------|---------------------------|
| **1** | **PVC `web‑pvc` → StorageClass `nfs‑sc` (implicit)** → **NFS Provisioner** | The PVC requests *“I need 5 Gi with RWX”*. The StorageClass (`nfs‑sc`) forwards that request to the NFS Provisioner. |
| **2** | **NFS Provisioner → NFS Server** | The provisioner creates (or re‑uses) a sub‑directory on the NFS Server and turns it into a PersistentVolume. |
| **3** | **Job Pod `upload‑content‑job` → PVC `web‑pvc`** | The Job mounts the PVC at `/data` and writes `index.html` into the shared NFS storage. |
| **4** | **Deployment Pod `web‑server` (nginx) → PVC `web‑pvc`** | The nginx Pod mounts the same PVC at `/usr/share/nginx/html`, so it immediately sees the file created by the Job. |
| **5** | **Deployment Pod `web‑server` → Service `web‑server‑svc`** | The Service selects Pods labeled `web-server` and forwards all HTTP traffic to them (port 80). |
| **6** | **Service `web‑server‑svc` → AWS ELB / public IP** | Since the Service is type `LoadBalancer`, AWS provisions an external load balancer with a public IP/DNS name. |
| **7** | **AWS ELB → Client (browser)** | The client accesses the public address; traffic reaches the ELB, which forwards requests to the nginx Pod serving content from NFS. |




## Setup
```bash
# 1. Configure AWS credentials
aws configure set default.region us-east-1
aws configure set default.output table
aws configure
# Check the credentials file
nano ~/.aws/credentials

# Verify AWS connection
aws sts get-caller-identity

# 2. Configure kubeconfig for EKS
aws eks update-kubeconfig --region <REGION> --name <CLUSTER>

# Check if nodes are ready
kubectl get nodes

# 3. Install Helm and add NFS Ganesha repo
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
helm version
helm repo add nfs-ganesha https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner
helm repo update

# 4. Create namespace for the application
kubectl create namespace nfs

# 5. Install NFS provisioner using Helm
helm install nfs nfs-ganesha/nfs-server-provisioner -n nfs -f helm/values.yaml

# Verify that the NFS provisioner pod is running
kubectl -n nfs get pods

# 6. Deploy application manifests (Deployment, Service, Job, PVC)
kubectl apply -f k8s/ -n nfs

# Check status of components
kubectl get deploy -n nfs
kubectl get pods -n nfs
kubectl get svc -n nfs
kubectl get pvc -n nfs web-pvc
kubectl get jobs -n nfs

# or check all resources
kubectl get all -n nfs
# 7. Test the application
ELB=$(kubectl get svc web-server-svc -n nfs -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ELB      # => <h1>Hello from NFS!</h1>
# or open the URL in a browser

```
## Clean up resources
```bash
# Delete the application components (Deployment, Job, Service, PVC)
kubectl delete -f k8s/ -n nfs

# Uninstall the NFS provisioner Helm release
helm uninstall nfs -n nfs

# Delete the dedicated namespace
kubectl delete namespace nfs
```


---

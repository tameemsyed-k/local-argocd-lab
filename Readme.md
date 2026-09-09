# �� ArgoCD & GitOps Lab (Local Kubernetes Environment)

Welcome to the **Local ArgoCD GitOps Lab**! This repository contains declarative ArgoCD `Application` Custom Resource Definitions (CRDs) and custom Helm charts to deploy **Nginx** and **Grafana** workloads onto a local Kubernetes cluster using GitOps continuous delivery.

---

## 📌 Repository Overview

- **Repository**: [`https://github.com/tameemsyed-k/local-argocd-lab.git`](https://github.com/tameemsyed-k/local-argocd-lab.git)
- **Target Cluster**: Local Kubernetes (Minikube, Kind, K3s, or Docker Desktop K8s)
- **ArgoCD Control Namespace**: `argocd`

### 📂 Directory Structure
```text
.
├── argo-apps/
│   ├── nginx-app.yaml                 # ArgoCD Application manifest for Nginx
│   └── grafana-app.yaml               # ArgoCD Application manifest for Grafana
├── grafana/                            # Custom Grafana Helm Chart (deployment, service, pvc)
├── nginx/                              # Custom Nginx Helm Chart (deployment, service)
└── Readme.md                           # Lab Documentation
```

---

## 🛠️ Step-by-Step Execution Guide

### Step 1: Install & Verify ArgoCD
1. Create the dedicated `argocd` namespace:
   ```bash
   kubectl create namespace argocd
   ```
2. Install ArgoCD:
   ```bash
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. Verify all ArgoCD microservice pods are `Running`:
   ```bash
   kubectl get pods -n argocd
   ```

---

### Step 2: Access ArgoCD & Retrieve Credentials
1. Decode the initial `admin` password:
   ```bash
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo ""
   ```
2. Forward the ArgoCD web server port to your local machine:
   ```bash
   kubectl port-forward service/argocd-server -n argocd 8080:443
   ```
3. Open your browser: **[https://localhost:8080](https://localhost:8080)**
   - **Username**: `admin`
   - **Password**: *(Decoded password from step 1)*

---

### Step 3: Local Environment Manifest Adaptations

The manifests in `argo-apps/` bridge this Git repo with your local cluster:

#### 1. ArgoCD Control Namespace
Set `metadata.namespace: argocd` so ArgoCD can discover the Application objects.

#### 2. Local Helm Overrides (`argo-apps/grafana-app.yaml`)
Overridden cloud settings directly inside the ArgoCD Application manifest:
- **`persistence.storageClass: local-path`** (Uses local volume provisioner instead of AWS `gp2`).
- **`service.type: NodePort`** (Accessible locally without AWS LoadBalancers).
- **`replicaCount: 1`** (Matches ReadWriteOnce single-mount PVC access mode).

---

### Step 4: Create Destination Namespaces & Deploy Apps

1. Create target workload namespaces:
   ```bash
   kubectl create namespace web-apps
   kubectl create namespace monitoring
   ```
2. Apply the ArgoCD Application manifests:
   ```bash
   kubectl apply -f argo-apps/ -n argocd
   ```
3. Verify application sync & health status:
   ```bash
   kubectl get application -n argocd
   ```

---

### Step 5: Verify Workloads in Kubernetes

- **Nginx Pods & Service**:
  ```bash
  kubectl get pods,svc -n web-apps
  ```
- **Grafana Pods, Service & PVC**:
  ```bash
  kubectl get pods,svc,pvc -n monitoring
  ```

---

### Step 6: Test GitOps Workflows & Self-Healing

#### A. Trigger Automated Self-Healing
Manually delete pods in Kubernetes to observe ArgoCD automatically repair the cluster back to match Git:
```bash
kubectl delete pod -n web-apps --all
```

#### B. Continuous Delivery Workflow
1. Make a local code change (e.g. modify replica count in `nginx/values.yaml`).
2. Commit & push to GitHub:
   ```bash
   git commit -am "Update nginx deployment configuration"
   git push
   ```
3. Watch ArgoCD detect the GitHub commit and update your cluster live!

---

## 🧹 Cleanup Instructions

To remove lab applications and workload namespaces:
```bash
kubectl delete application nginx-app grafana-app -n argocd
kubectl delete namespace web-apps monitoring
```

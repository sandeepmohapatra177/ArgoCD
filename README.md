# ArgoCD Basics

## 2.1 Why ArgoCD for GitOps?

* **Purpose-built for Kubernetes**: ArgoCD is a Kubernetes controller, native to the ecosystem.
* **Declarative + GitOps**: Uses Git repos as the source of truth.
* **Multi-cluster ready**: Manage multiple clusters from one ArgoCD instance.
* **UI + CLI + API**: Rich user interface and automation APIs.
* **Security**: In-cluster controller, supports RBAC, SSO, audit logs.
* **Ecosystem**: Part of Argo Project (with Workflows, Rollouts, Events).

ArgoCD makes GitOps adoption beginner-friendly and production-grade.

---

## 2.2 ArgoCD vs FluxCD vs Jenkins X

| Tool          | Strengths                                                             | Weaknesses                                                         |
| ------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **ArgoCD**    | UI, multi-cluster support, app-centric model, notifications, rollouts | Slightly heavier footprint, mainly K8s-focused                     |
| **FluxCD**    | Lightweight, GitOps-first, good Helm/Kustomize support                | No UI (only CLI/alerts), app model less visual                     |
| **Jenkins X** | GitOps + CI/CD combined, integrates with Tekton pipelines             | Complex, heavy, opinionated, less adoption compared to ArgoCD/Flux |

---

## 2.3 ArgoCD Architecture

![ArgoCD Architecture](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

**Core components:**

1. **API Server**

   * Frontend for CLI/UI/REST/gRPC.
   * Handles RBAC, authentication, and API requests.

2. **Repo Server**

   * Connects to Git repos.
   * Fetches manifests (raw YAML, Helm templates, Kustomize overlays).
   * Caches and serves manifests to the Application Controller.

3. **Application Controller**

   * The brain of ArgoCD.
   * Continuously compares desired state (from Git) with live state (in cluster).
   * Applies changes (sync), monitors health, handles rollbacks.

4. **UI / CLI**

   * **UI**: Visual dashboard for apps, sync status, health.
   * **CLI (`argocd`)**: Automation and scripting support.

---

## 2.4 Key ArgoCD Concepts

### 1. **Application**

* The core ArgoCD object.
* Links a Git repo → path → cluster → namespace.
* Represents a deployed app.

### 2. **Project**

* Logical grouping of Applications.
* Defines RBAC, repo access, cluster targets.
* Example: `team-a` project can only deploy apps to `team-a-*` namespaces.

### 3. **Repositories**

* Git repos registered with ArgoCD.
* Can store Helm charts, Kustomize overlays, or raw manifests.

### 4. **Health Status**

* **Healthy**: Desired state = live state.
* **Degraded**: App not working (e.g., CrashLoopBackOff).
* **Progressing**: App is deploying.
* **Missing**: Resource expected but not found.

### 5. **Rollbacks**

* ArgoCD keeps app history.
* You can rollback to a previous healthy revision.

### 6. **Auto-Healing**

* If drift is detected (e.g., a pod deleted manually), ArgoCD restores it.
* Demo: `kubectl delete pod <pod>` → ArgoCD recreates it.

### 7. **Sync & Sync Policies**

* **Manual**: User clicks/CLI `argocd app sync`.
* **Automatic**: ArgoCD auto-applies Git changes.
* **Sync Waves**: Control order of resource sync (e.g., DB before App).
* **Sync Hooks**: Custom scripts/jobs at sync phases:

  * **PreSync** → before sync.
  * **Sync** → during sync.
  * **PostSync** → after sync.
  * **SyncFail** → if sync fails.

* **Flags**:

  * `--prune` → delete resources not in Git.
  * `--replace` → force re-create instead of patch.

### 8. **Sync Options**

* **Skip Schema Validation**: Skip Kubernetes schema validation.
* **Auto-Create Namespace**: If namespace doesn’t exist, create it.
* **Prune Last**: Delete resources at end of sync (safer order).
* **Apply Out of Sync Only**: Only apply changed resources.
* **Respect Ignore Differences**: Don’t overwrite ignored fields.
* **Server-Side Apply**: Use Kubernetes server-side apply for merges.

---

## 2.5 Summary

* GitOps makes Git the source of truth.
* ArgoCD implements GitOps with continuous reconciliation.
* Core components: API Server, Repo Server, App Controller, UI/CLI.
* Key concepts: Applications, Projects, Health, Sync, Rollbacks, Auto-healing.
* Compared to Flux and Jenkins X, ArgoCD provides the most visual and app-centric GitOps experience.


---

Happy Learning!




# ArgoCD Setup and Installation

Let's see how we can Setup & Install ArgoCD (UI and CLI) and access via the browser.

---

# Prerequisites

Before starting, ensure you have the following installed on your system:

1. **Docker** → Required for Kind to run containers as cluster nodes.

   ```bash
   sudo apt-get update
   sudo apt install docker.io -y
   sudo usermod -aG docker $USER && newgrp docker
   docker --version

   docker ps
   ```

2. **Kind (Kubernetes in Docker)** → To create the cluster.

   ```bash
   kind version
   ```

   [Install Guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

3. **kubectl** → To interact with the cluster.

   ```bash
   kubectl version --client
   ```

   [Install Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

4. **Helm (for Helm-based installation)**

   ```bash
   helm version
   ```

   [Install Guide](https://helm.sh/docs/intro/install/)

---

> [!IMPORTANT]
> 
> You can either follow the below steps or directly run the script [setup_argocd.sh](./setup_argocd.sh)
> 
> The script will create **kind cluster** and **Installs ArgoCD UI and CLI** based on your choice (using HELM or manifest)
> 
> But before using this guide or `setup_argocd.sh`, make sure you replace the `172.31.19.178` address with your EC2 instance private ip in Cluster config for `apiServerAddress`

---

# Step 1: Create Kind Cluster

Save your cluster config as `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "172.31.19.178"   # Change this to your EC2 private IP (run "hostname -I" to check or from your EC2 dashboard)
  apiServerPort: 33893
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
```

> Why `apiServerAddress` & `apiServerPort` in kind config?
→ To ensure each kind cluster API server is reachable from the ArgoCD pods. This avoids conflicts (since kind defaults to random localhost ports).

Create the cluster:

```bash
kind create cluster --name argocd-cluster --config kind-config.yaml
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

---

#  Step 2: Install ArgoCD

We’ll cover **two professional installation methods**.

---

## **Method 1: Install ArgoCD using Helm** (recommended for customization/production)

### 1. Add Argo Helm repo

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Create namespace

```bash
kubectl create namespace argocd
```

### 3. Install ArgoCD

```bash
helm install argocd argo/argo-cd -n argocd
```

### 4. Verify installation

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

### 5. Access the ArgoCD UI

Port-forward the service:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

Now open → **[https://<instance_public_ip>:8080](https://<instance_public_ip>:8080)**

### 6. Get initial admin password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* Username: `admin`
* Password: (above output)

---

## **Method 2: Install ArgoCD using Official Manifests (kubectl apply)**

(fastest for demos & learning)

### 1. Create namespace

```bash
kubectl create namespace argocd
```

### 2. Apply ArgoCD installation manifest

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Verify installation

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

### 4. Expose ArgoCD server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

Access → **[https://<instance_public_ip>:8080](https://<instance_public_ip>:8080)**

### 5. Get initial password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* Username: `admin`
* Password: (above output)

---

# Step 3: Install ArgoCD CLI (Ubuntu/Linux)

ArgoCD server runs inside Kubernetes, but to interact with it from the terminal you need the **ArgoCD CLI (`argocd`)**.  
This is separate from the server installation.

### 1. Install ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 2. Verify installation

```bash
# Verify installation
argocd version --client
```

### 3. Login to ArgoCD CLI

```bash
argocd login <instance_public_ip>:8080 --username admin --password <initial_password> --insecure
```

> Note: The --insecure flag is required when using port-forward with self-signed TLS certs.
For production, you’d configure proper TLS certs (then --insecure is not needed).

### 4. Get user info

```bash
argocd account get-user-info
```

---

#  Helm vs Manifest Installation

| Feature         | Helm Install (Method 1)     | Manifests (Method 2)         |
| --------------- | --------------------------- | ---------------------------- |
| **Flexibility** | High (override values.yaml) | Low (default configs only)   |
| **Ease of Use** | Requires Helm               | Works with just kubectl      |
| **Best for**    | Production & customization  | Quick demo / lab environment |

---

# Professional Best Practices

* For **local demo/testing** → use **kubectl apply**.
* For **production or enterprise** → use **Helm** (better upgrades & customization).
* Always **separate namespaces** (don’t install into `default`).
* Store **Application CRDs** in Git repos (GitOps best practice).




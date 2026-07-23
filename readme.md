### 4.1 Folder structure

```
kustomize-lab/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

**Why this structure:** `base/` holds the "canonical" definition of the app. Each folder under `overlays/` represents one target environment, and contains only the *differences* from base plus a `kustomization.yaml` that ties everything together.

### 4.2 Base manifests

`base/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```
This is a plain, ordinary Deployment — nothing Kustomize-specific about it. That's intentional: Kustomize works on standard Kubernetes YAML, unlike Helm which uses its own templating syntax.

`base/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

`base/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```
**What this does:** a `kustomization.yaml` file is what makes a folder a "kustomization" that `kubectl -k` or `kustomize` can process. `resources:` lists the plain YAML files this kustomization bundles together.

### 4.3 Dev overlay (namespace + name prefix only)

`overlays/dev/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
namePrefix: dev-
resources:
  - ../../base
```
**Field-by-field explanation:**
- `resources: [../../base]` — rather than listing raw YAML files, this overlay points at the *base folder itself*, pulling in everything it defines.
- `namespace: dev` — a transformer that rewrites every resource in the base to live in the `dev` namespace, without you having to edit the base YAML at all.
- `namePrefix: dev-` — another transformer, prepending `dev-` to every resource's name (so the Deployment becomes `dev-web`). This keeps resources visually identifiable and avoids name collisions if multiple overlays ever shared a namespace.

### 4.4 Prod overlay (more replicas + patch)

`overlays/prod/replica-patch.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
```
**What this is:** a strategic merge patch — a *partial* Deployment manifest containing only the fields you want to override. Kustomize merges this on top of the base Deployment at build time, leaving every other field untouched.

`overlays/prod/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
namePrefix: prod-
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
    target:
      kind: Deployment
      name: web
```
**Field-by-field explanation:**
- `patches[].path` — the patch file to apply.
- `patches[].target` — which resource(s) the patch applies to, matched by `kind` and `name` (matching the *base* resource's name, before `namePrefix` is applied).

### 4.5 Try it

```bash
kubectl create namespace dev
kubectl create namespace prod
```

```bash
# Preview the rendered YAML without applying anything
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
```
**What this does:** `kubectl kustomize <folder>` runs the overlay/base merge process and prints the *final*, fully-resolved YAML to your terminal, without touching the cluster. This is the single most useful Kustomize debugging habit — always preview before applying.

```bash
# Apply each overlay
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```
**Flag breakdown:**
- `-k` — tells `kubectl apply` to treat the given path as a kustomization directory (run it through the Kustomize build process) rather than a plain YAML file.

```bash
kubectl get deploy -n dev
kubectl get deploy -n prod
```

**What to notice:** the *same* base Deployment produced a 1-replica `dev-web` Deployment in the `dev` namespace, and a 4-replica `prod-web` Deployment in `prod` — with zero duplicated YAML and no templating language involved, just overlays and patches.

### 4.6 Cleanup

```bash
kubectl delete -k overlays/dev
kubectl delete -k overlays/prod
```
**What this does:** `kubectl delete -k <folder>` deletes exactly the set of resources that `kubectl apply -k <folder>` would have created — the reverse operation, computed the same way.

---

## 5 GitHub Actions deployment pipeline for AKS

This repo can be deployed to an existing AKS cluster from GitHub using a GitHub Actions workflow.

### 5.1 Workflow file

Place the workflow at `.github/workflows/aks-kustomize-deploy.yml`.

The sample workflow:

- checks out the repository
- logs in to Azure using `AZURE_CREDENTIALS`
- sets the AKS context for the existing cluster using `az aks get-credentials`
- previews the rendered Kustomize YAML
- applies `kustomize-lab/overlays/dev`
- confirms the deployed Deployment and Service

### 5.2 Required GitHub secrets

Configure these secrets in the repository settings:

- `AZURE_CREDENTIALS` - JSON credentials for an Azure service principal
- `AKS_RESOURCE_GROUP` - the AKS cluster resource group
- `AKS_CLUSTER_NAME` - the AKS cluster name

### 5.3 Create authentication for the pipeline

From a machine that can access your Azure subscription, run:

```bash
az login
az account set --subscription <subscription-id>
az ad sp create-for-rbac \
  --name "github-actions-aks-deploy" \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<aks-resource-group>/providers/Microsoft.ContainerService/managedClusters/<aks-cluster-name> \
  --sdk-auth
```

Copy the JSON output and save it into GitHub as `AZURE_CREDENTIALS`.

### 5.4 Required permissions

The service principal must have permission to:

- access the AKS cluster resource via Azure RBAC
- request cluster credentials via `az aks get-credentials`
- deploy Kubernetes resources with `kubectl apply`

The `Azure Kubernetes Service Cluster User Role` is the least-privilege Azure role for this pipeline. If you need broader repository or resource-group-level deployment permissions, use `Contributor` scoped to the AKS resource group.

### 5.5 AKS cluster RBAC notes

If the cluster uses Azure AD-managed Kubernetes RBAC, the service principal may also need a Kubernetes cluster role binding that allows it to apply resources.

For example, run once from an administrator session:

```bash
kubectl create clusterrolebinding github-actions-aks-deploy \
  --clusterrole=cluster-admin \
  --user <service-principal-app-id>
```

### 5.6 Deploying other overlays

To deploy a different overlay, update the workflow file path in the `kubectl apply -k` step to:

```yaml
kubectl apply -k kustomize-lab/overlays/prod
```

If you want to deploy both environments from GitHub, add separate workflow jobs or use `workflow_dispatch` inputs for `dev` and `prod`.

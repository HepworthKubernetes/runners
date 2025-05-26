# Creating and Deploying GitHub Runners

## Access Tokens

The deployment requires an `ACCESS_TOKEN` to be set as a secret in the `github-runners` namespace.

### Required scopes for the runner:

#### Personal repository
- `repo` (all)
- `workflow`
- `admin:repo_hook`, `read:repo_hook`
- `admin:public_key`, `read:public_key`
- `notifications`

#### Organization
- `admin:org` (all)
- `admin:org_hook`

Add the token to the cluster:

```bash
kubectl create secret generic github-runner \
    --from-literal=ACCESS_TOKEN=$(wl-paste) \
    -n github-runners
```

### Required scopes for the Dockerfile build:

- `repo` (all)
- `read:packages`
- `write:packages`

Add the token for the image pull:

```bash
kubectl create secret docker-registry ghcr-creds \
    --docker-server=ghcr.io \
    --docker-username=travishepworth \
    --docker-password=$(wl-paste) \
    -n github-runners
```

## Deploying the Runners

On a new cluster, or if the runners get deleted, they need a manual start. Ideally your image is still there for deployment - if not you have to jumpstart it instead of just deploy a new one.

### Project Structure

NOTE: this is for a single dir structure, we have moved to using an org.

```
├── base
│   └── github-runner
│       ├── deployment.yaml
│       ├── kustomization.yaml
│       └── README.md
├── cluster
│   ├── namespaces
│   │   ├── github-runners
│   │   │   ├── kustomization.yaml
│   │   │   ├── namespace.yaml
│   │   │   └── rbac
│   │   │       ├── deploy-sa.yaml
│   │   │       ├── kustomization.yaml
│   │   │       ├── runner-rb.yaml
│   │   │       └── runner-sa.yaml
│   │   └── terraria
│   │       └── kustomization.yaml
│   └── rbac
│       └── github-runner
│           ├── gh-runner-deploy-rb.yaml
│           └── kustomization.yaml
├── images
│   └── github-runner
│       └── Dockerfile
└── README.md
```

### 1. Create the namespace and RBAC

```bash
kubectl apply -k cluster/namespaces/github-runners
```

This creates:

- `github-runners` namespace
- `gh-runner-deploy-sa` (privileged deploy service account)
- `gh-runner-sa` (standard service account)
- `runner-rb` role binding for the `gh-runner-sa`

The deploy runner gets elevated permissions; the normal runner is sandboxed for non-cluster tasks.

### 2. Deploy the runners

```bash
kubectl apply -k k8s/overlays/runner-a
# deployment.apps/gh-runner created

kubectl apply -k k8s/overlays/runner-b
# deployment.apps/gh-runner created

kubectl apply -k k8s/overlays/runner-deploy
# deployment.apps/gh-runner created
```

Note: this assumes the required Docker registry is available for the deployment container (which needs `kubectl`).  
TODO: Document the image build and link it here.

### 3. Create the target namespace (e.g., for workloads)

```bash
kubectl apply -k cluster/namespaces/terraria
```

This ensures the deploy runner has access, while the normal runner does not. The role binding is scoped accordingly.

### Access Verification

```bash
kubectl auth can-i create deployment \
  --as=system:serviceaccount:github-runners:gh-runner-sa \
  --namespace=terraria
# no

kubectl auth can-i create deployment \
  --as=system:serviceaccount:github-runners:gh-runner-deploy-sa \
  --namespace=terraria
# yes
```

## Notes

Configure runner tags carefully so jobs are dispatched appropriately to deploy vs. normal runners. Use tags in your GitHub Actions workflows to target the right type of runner based on access needs and permissions.

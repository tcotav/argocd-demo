# ArgoCD ApplicationSet Demo

This repository demonstrates how to use ArgoCD ApplicationSet to deploy multiple applications with different configurations and labels from YAML files.

## Overview

This demo shows how labels and parameters from YAML configuration files are automatically passed into a Helm chart and applied to Kubernetes resources.

## Repository Structure

```
.
├── Chart.yaml                  # Helm chart definition
├── values.yaml                 # Default Helm values
├── templates/
│   └── configmap.yaml         # Kubernetes ConfigMap template with labels
├── configs/                    # Application configuration files
│   ├── app-dev.yaml           # Dev environment config
│   ├── app-staging.yaml       # Staging environment config
│   └── app-prod.yaml          # Production environment config
└── applicationset/
    └── sample-app-applicationset.yaml  # ApplicationSet definition
```

## How It Works

1. **Configuration Files** (`configs/app-*.yaml`): Each YAML file defines an application with:
   - Application name
   - Environment
   - Team
   - Version
   - Custom labels (owner, costCenter, criticality, etc.)

2. **ApplicationSet**: Reads all `app-*.yaml` files from the `configs/` directory and creates an ArgoCD Application for each one.

3. **Helm Chart**: Uses the values from the config files to:
   - Create a ConfigMap with labels
   - Apply custom labels to the Kubernetes resources
   - Set configuration data

## Setup Instructions

### Prerequisites

- Minikube running
- ArgoCD installed in your cluster
- kubectl configured to access your cluster

### Step 1: Install ArgoCD (if not already installed)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD to be ready:

```bash
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Step 2: Access ArgoCD UI (optional)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Access the UI at https://localhost:8080

### Step 3: Update the ApplicationSet with your GitHub repository

Edit [applicationset/sample-app-applicationset.yaml](applicationset/sample-app-applicationset.yaml) and replace `YOUR-USERNAME` with your GitHub username in both `repoURL` fields.

### Step 4: Push to GitHub

```bash
git add .
git commit -m "Add ArgoCD ApplicationSet demo"
git push origin main
```

### Step 5: Deploy the ApplicationSet

```bash
kubectl apply -f applicationset/sample-app-applicationset.yaml
```

### Step 6: Verify the Applications

Check the ApplicationSet:

```bash
kubectl get applicationset -n argocd
```

Check the generated Applications:

```bash
kubectl get applications -n argocd
```

You should see three applications:
- `frontend-app-dev`
- `frontend-app-staging`
- `frontend-app-prod`

## Viewing the Labels

Once the applications are synced, you can verify that the labels from the YAML files were applied:

### Check ConfigMaps with labels:

```bash
# Dev environment
kubectl get configmap frontend-app-config -n dev --show-labels

# Staging environment
kubectl get configmap frontend-app-config -n staging --show-labels

# Production environment
kubectl get configmap frontend-app-config -n prod --show-labels
```

### View detailed labels:

```bash
kubectl get configmap frontend-app-config -n dev -o yaml | grep -A 10 labels
kubectl get configmap frontend-app-config -n staging -o yaml | grep -A 10 labels
kubectl get configmap frontend-app-config -n prod -o yaml | grep -A 10 labels
```

### Expected Output

Each ConfigMap will have different labels based on the config file:

**Dev** (from `configs/app-dev.yaml`):
```yaml
labels:
  app: frontend-app
  environment: dev
  team: frontend
  version: "1.0.0"
  owner: alice
  costCenter: engineering
  criticality: low
```

**Staging** (from `configs/app-staging.yaml`):
```yaml
labels:
  app: frontend-app
  environment: staging
  team: frontend
  version: "1.2.0"
  owner: alice
  costCenter: engineering
  criticality: medium
```

**Production** (from `configs/app-prod.yaml`):
```yaml
labels:
  app: frontend-app
  environment: prod
  team: frontend
  version: "2.0.0"
  owner: alice
  costCenter: engineering
  criticality: high
  compliance: pci-dss
```

## Adding New Environments

To add a new environment, simply create a new YAML file in the `configs/` directory:

```bash
cat > configs/app-qa.yaml <<EOF
appName: frontend-app
environment: qa
team: frontend
version: "1.1.0"
namespace: qa
customLabels:
  owner: bob
  costCenter: qa-dept
  criticality: medium
EOF
```

Commit and push:

```bash
git add configs/app-qa.yaml
git commit -m "Add QA environment"
git push origin main
```

ArgoCD will automatically detect the new file and create a new Application!

## Cleanup

To remove all applications:

```bash
kubectl delete applicationset sample-app-set -n argocd
kubectl delete namespace dev staging prod qa
```

## Key Concepts Demonstrated

1. **Git Generator**: ApplicationSet uses the Git generator to scan for config files
2. **Parameter Injection**: Values from YAML files are injected into Helm values
3. **Label Propagation**: Labels from config files are applied to Kubernetes resources
4. **Dynamic Application Creation**: New applications are created automatically when new config files are added
5. **Environment-specific Configuration**: Each environment has its own config with different labels and values

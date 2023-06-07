# Introduction 

GitOps Repository Example with ArgoCD and Kustomize (with a pinch of Helm)

# Getting Started

## Requirements
- kubectl
- yq
- kustomize >= v5.0.0
- helm >= v3.x

## Bootstrapping a Cluster
1. **Install ArgoCD's dependencies**
   - ArgoCD itself may depend on certain apps (e.g. external-secrets, cilium)
   - If so, vendor them into `apps/` and apply them as necessary.
     A `kustomization.yaml` is usually enough to pull in both helm charts and raw installation manifests.
2. **Install ArgoCD**
   - Create the namespace `argocd`
   - (Optional) When using a private repo, create a secret named `gitops-repo` for ArgoCD to access this repository, with the label `argocd.argoproj.io/secret-type: repository`.
    
     You can also utilize External Secrets Operator to pull the secret from external sources.
     ```yaml
     apiVersion: v1
     kind: Secret
     metadata:
       labels:
         argocd.argoproj.io/secret-type: repository
       name: gitops-repo
       namespace: argocd
     type: Opaque
     stringData:
       enableLfs: "false"
       url: "git@github.com:kloia/ArgoCD-EKS-Bootstrapper.git"
       sshPrivateKey: |
       -----BEGIN OPENSSH PRIVATE KEY-----
       ...
       -----END OPENSSH PRIVATE KEY-----

     ```
   - Apply the ArgoCD Manifest
     ```shell
     kustomize build apps/argocd/base | kubectl apply -f -
     ```
3. **Apply the ArgoCD Application representing the cluster (App of Apps)**
   - Find and apply the cluster specific, patched "app of apps" manifest, which will install everything that the cluster needs
     ```shell
     kustomize build clusters/overlays/cluster-foo | yq '. | select(.kind == "Application" and .metadata.name == "app-of-apps")' | kubectl apply -f -
     ```

# Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for project structure and notes.

# Contributing

## Creating and Deploying an App

1. Create a new app directory and the required bases/overlays (e.g. `apps/<app-name>`).
2. Create the ArgoCD Application for clusters.
 
   Warning: Do not enable autosync for Applications authored for the first time. Mistakes happen, double check its diff through the ArgoCD UI first.
3. Open a Pull Request for review.
4. Merge to main.
5. (Optional) Manually sync deployment through ArgoCD UI or CLI if autosync is not enabled for the application.


# Possible Improvements
- Showcasing single ArgoCD instance managing multiple clusters. Current structure deploys a separate ArgoCD instance per cluster.
- Ready to use App of common resources. Things like ESO and Kyverno etc, can be referenced from this repo, instead of copying the structure of the entire repo.

# Recommended Reading

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)

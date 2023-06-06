# Architecture

This project uses [kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/) for managing kubernetes manifests,
and Helm charts for some off-the-shelf apps. Deployments and app synchronizations are handled by ArgoCD in each cluster.

## Apps

The `apps/` directory contains kustomizations for both bespoke (user authored) and off-the-shelf "apps".

```
apps/
└── <app-name>/
    ├── base/                        # Base resources
    │   ├── resources/
    │   │   ├── foo.yaml
    │   │   ├── bar.yaml
    │   │   └── ...
    │   └── kustomization.yaml
    └── overlays/                    # Environment specific kustomizations
        ├── dev/
        │   ├── kustomization.yaml   # References ../../base and patches base manifests
        │   └── ...
        ├── uat/
        │   ├── kustomization.yaml
        │   └── ...
        └── ...                      # Overlay name does not have to be namespaces
```

### Notable apps
- `apps/kuard` showcases a bespoke app with multiple configurations
- `apps/external-secrets-operator` showcases "vendoring" in a Helm chart using kustomize.

### Notes
- **An app does not have to be a kubernetes deployment**

  It is essentially just a bundle of manifests that can be manipulated and
  applied together in a single `kubectl apply -f -`.

- **An app's `base/` directory is not guaranteed to be valid kubernetes manifests by itself**.

  Solely deploying a base may fail if you try to deploy it without a correct overlay. This is mostly for apps that can't
  function outside the context of an "environment".

- **Environment Promotion and Overlay Names**

  Some bespoke applications may need a particular structure of overlays for version promotions between environments to work.

  e.g. A pipeline that updates versions by copying values from `overlays/dev` to `overlays/staging` and committing the change back to the repo.

- **Bundle related resources together as much as possible**

  There are certain exceptions:

  - **Multiple apps depend on shared resources in another app**

    While resources like Ingress are feasible to bundle with an app, a config map consumed by multiple apps is better left
    in its dedicated app. Either leave as is, or try to refactor the contents of the config map.

  - **Apps that define a CustomResourceDefinition and use the CustomResources in the same app does not work**

    The resulting manifests of an app are applied with a single `kubectl apply -f -`, all at once (which kubernetes cannot do with the case of CRDs and CRs).

    You need to separate it into 2 apps, one producing the CRD, one using the CR.

## Clusters

The `clusters/` directory contains ArgoCD Applications that make up clusters.
Each ArgoCD Application either points to an app directory under `apps/` or a Helm chart.

```
clusters/
├── base/                                      # Common Applications for every cluster
│   ├── resources/
│   │   └── applications/
│   │       ├── argocd-app-1.yaml              # Defines an App that resides in `apps/`
│   │       └── argocd-app-2.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── cluster-foo/                           # Cluster name
│   │   ├── resources/
│   │   │   └── applications/
│   │   │       └── cluster-specific-app.yaml  # Cluster specific application
│   │   └── kustomization.yaml                 # References ../../base, patches cluster specific values
│   ├── <cluster-name>/
│   └── ...
└── components/
    └── ...                                    # Helper Kustomize Components for ArgoCD Options

```

### Notes

- **Applications put under `clusters/base` will be synced by ALL clusters.**

  If you need to deploy an Application to multiple clusters but not all of them, copy them across overlays.

- **Application Namespaces**

  ArgoCD requires Application definitions to be in the namespace `argocd`. To define the namespace of the applied
  manifests, use `.spec.destination.namespace`.

- **ApplicationSet**

  ApplicationSet Custom Resource can be used for applications with multiple environments in the same cluster.

# Flux Repo Structure - Fleet of Clusters

## Overview

## Prerequisites
### Kubernetes clusters
For example let's use two local clusters for our Fleet.
I'm going to use `kind` tool to create clusters. [[doc](https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster)]

Non-prod cluster:
```
kind create cluster --image kindest/node:v1.23.4 --name non-prod
```

Prod cluster:
```
kind create cluster --image kindest/node:v1.23.4 --name prod
```

Check that clusters are up and running
```
kubectl --context kind-non-prod get nodes

NAME                     STATUS   ROLES                  AGE     VERSION
non-prod-control-plane   Ready    control-plane,master   3m54s   v1.23.4
```

```
kubectl --context kind-prod get nodes

NAME                 STATUS   ROLES                  AGE   VERSION
prod-control-plane   Ready    control-plane,master   84s   v1.23.4
```

### Bootstrap the clusters
See the [doc](https://fluxcd.io/docs/installation/)

Create PAT (Personal Access Token) [[doc](https://docs.github.com/en/enterprise-server@3.4/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)].
```
export GITHUB_TOKEN=<your-token>
```

Bootstrap non-prod cluster
```
kubectl config set-context kind-non-prod
flux bootstrap github \
  --owner=vterdunov \
  --repository=frs-k8s-fleet \
  --path=clusters/non-prod \
  --personal
```

Bootstrap prod cluster
```
kubectl config set-context kind-prod
flux bootstrap github \
  --owner=vterdunov \
  --repository=frs-k8s-fleet \
  --path=clusters/prod \
  --personal
```

### Recap
At this moment we have two Kubernetes clusters with two Flux installation pointed to `frs-k8s-fleet` repository.
- Non-prod Flux is tracking `./clusters/non-prod` path.
- Prod Flux is tracking `./clusters/prod` path.

Repo structure for this momemt is available in `stage-0` git branch.

## Flux Repo Structure
On the `./clusters/<cluster>` level we're going to use two directories for our workloads.
- `./clusters/<cluster>/infrastructure` for cluster wide infrastructure components. Such as monitoring, logs collectors, some cluster wide Databases. In other words everything that cluster admin should deploy to clusters.
- `./clusters/<cluster>/tenants` for a company teams workloads. Microservices deployments, dashboards and so on.

```
mkdir -p ./clusters/{prod,non-prod}/{infrastructure,tenants}
mkdir -p ./infrastructure/base
mkdir -p ./infrastructure/overlays/{prod,non-prod}
```


### Prometheus Operator Custom Resource Definitions
Almost every popular workload in kubernetes provdes metrics in Prometheus format. Even kubernetes components itself. It convinient to use Pormetheus Opeartor or VictroriaMetrics Operator to dynamicaly add and operate targets to Prometheus or VictroiaMetrics.

We will use `ServiceMonitor` and `PodMonitor` Custom Resources. A lot of popular HelmChart allows to create `ServiceMonitor` by just adding a few lines in chart values. E.g. nginx ingress helm chart [values](https://github.com/kubernetes/ingress-nginx/blob/6d9a39eda7b180f27b34726d7a7a96d73808ce75/charts/ingress-nginx/values.yaml#L678).  
To be able to create a Custom Resource in Kubernetes need to create Custom Resource Definition. Therefore a lot of cluster infrastructure components **depends on** `Prometheus Operator Custom Resource Definitions`.

Let's define them on Gitops way.

#### Flux Kustomize
First of all create Flux Kustomization that point to `./infrastructure/<cluster>/prometheus-operator-crds`. Let's try from non-prod cluster.  
Create [clusters/non-prod/infrastructure/prometheus-operator-crds.yaml](clusters/non-prod/infrastructure/prometheus-operator-crds.yaml)


> See Flux Kustomize recommended settings [[doc](https://fluxcd.io/docs/components/kustomize/kustomization/#recommended-settings)]

#### Base Kustomization
We're going to use kustomization overlays [[doc](https://kustomize.io/)] to manage our Kubernetes Resources. But first we need to create `base` for our future overlays.
```
mkdir -p ./infrastructure/base/prometheus-operator-crds
cd ./infrastructure/base/prometheus-operator-crds
```

Prometheus Operator CRDs available on Github https://github.com/prometheus-operator/prometheus-operator/  
Get latest tag and download CRDs to a directory with name equal as the git tag.
For this momemnt latest tag is `v0.56.0`
```
export PROM_GIT_TAG='v0.56.0'

mkdir ${PROM_GIT_TAG} && cd ${PROM_GIT_TAG}

wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagerconfigs.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_podmonitors.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_probes.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_prometheusrules.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/${PROM_GIT_TAG}/example/prometheus-operator-crd/monitoring.coreos.com_thanosrulers.yaml

```

Generate `kustomization.yaml` that simplify to work with this whole `v0.56.0` directory in future. We will just point to the directory in `non-prod` overlay.
```
kustomize create --recursive --autodetect .
```

#### Overlay Kustomization
Go to repository root. Then create overlay directory
```
mkdir -p ./infrastructure/overlays/non-prod/prometheus-operator-crds
cd ./infrastructure/overlays/non-prod/prometheus-operator-crds
```
Create `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../../base/prometheus-operator-crds/v0.56.0

```

Validate that flux can build resources. Since Flux v0.29 flux cli allow to build manifests locally
```
flux build kustomization prometheus-operator-crds --path . --kustomization-file ../../../../clusters/non-prod/infrastructure/prometheus-operator-crds.yaml
```

or using `kustomize` binary.
```
kustomize build --load-restrictor=LoadRestrictionsNone --reorder=legacy .
```

### Cert Manager

#### Flux Kustomiation
Now let's to know how to manage dependencies with a CRD and CR on cert-manager example.  
Cert manager is a very popular kubernetes controller which has their own Custom Resources and Custom Resources Definitions.
We're going to deploy cert manager CRDs and deployments. And tell Flux to install CR only after the CRDs and Deployments will be successfully reconciled. To do that we create two different Flux Kustomization and one will be **depends** on of the other.

Create `./clusters/non-prod/cert-manager.yaml`

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: cert-manager-controller
  namespace: flux-system
spec:
  suspend: false
  interval: 60m
  wait: true
  timeout: 3m
  retryInterval: 2m
  prune: true
  force: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/overlays/non-prod/cert-manager/controller
  dependsOn:
    - name: prometheus-operator-crds

---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: cert-manager-cluster-issuers
  namespace: flux-system
spec:
  suspend: false
  interval: 60m
  wait: true
  timeout: 3m
  retryInterval: 2m
  prune: true
  force: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/overlays/non-prod/cert-manager/cluster-issuers
  dependsOn:
    - name: cert-manager-controller
```

As you notice `cert-manager-cluster-issuers` Flux Kustomization depends on `cert-manager-controller` which depends on `prometheus-operator-crds`. Therefore when it's time for the next reconcilation of the `cert-manager-cluster-issuers` Flux will check that `prometheus-operator-crds` and `cert-manager-controller` are ready.

#### Base Kustomization
We're going to use kustomization overlays [[doc](https://kustomize.io/)] to manage our Kubernetes Resources. But first we need to create `base` for our future overlays.
```
mkdir -p ./infrastructure/base/cert-manager/{controller,cluster-issuers}
```
**Controller**  
Prepare base controller manifests
See `./infrastructure/base/cert-manager/controller` manifests in the repo.

**Cluster Issuers CR**  
Prepare cluster-issuers manifests
See `./infrastructure/base/cert-manager/cluster-issuers` manifests in the repo.


#### Overlay Kustomization
Go to repository root. Then create overlay directories
```
mkdir -p ./infrastructure/overlays/non-prod/cert-manager/{controller,cluster-issuers}
```

**Controller**  
See `./infrastructure/overlays/non-prod/cert-manager/controller` manifests in the repo.
```
kustomize build --load-restrictor=LoadRestrictionsNone --reorder=legacy .
```

**Cluster Issuers CR**  
See `infrastructure/overlays/non-prod/cert-manager/cluster-issuers` manifests in the repo.
```
kustomize build --load-restrictor=LoadRestrictionsNone --reorder=legacy .
```

### Recap
At this moment we have two Kubernetes clusters with two Flux installation pointed to `frs-k8s-fleet` repository.
- Non-prod Flux is tracking `./clusters/non-prod` path.
- Prod Flux is tracking `./clusters/prod` path.

- `infrastructure/overlays/non-prod`  
We have deployed Prometheus Operator CRDs and Cert Manager in Gitops way.

```
flux tree ks flux-system --compact

Kustomization/flux-system/flux-system
├── Kustomization/flux-system/cert-manager-cluster-issuers
├── Kustomization/flux-system/cert-manager-controller
│   ├── HelmRelease/cert-manager/cert-manager
│   └── HelmRepository/cert-manager/cert-manager
├── Kustomization/flux-system/prometheus-operator-crds
└── GitRepository/flux-system/flux-system
```

Repo structure for this moment is available in `stage-1` git branch.

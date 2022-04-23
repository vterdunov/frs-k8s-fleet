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

Check that cluster are up and running
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

### Bootsrtap the clusters
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
At this moment we have two Kubernetes clusters with two Flux instalaltion pointed to `frs-k8s-fleet` repository.
Non-prod Flux is tracking `./clusters/non-prod` path.
Prod Flux is tracking `./clusters/prod` path.

## Flux Repo Structure


```bash
# ns
kubectl create namespace argo-rollouts

# quick start manifest
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/download/v1.8.3/install.yaml

# verify
kubectl get pods -n argo-rollouts
```


Install Rollouts kubectl Plugin

Unlike Argo CD and Argo Workflows, Argo Rollouts uses a kubectl plugin as its CLI client.

```bash
# install
brew install argoproj/tap/kubectl-argo-rollouts

# verify
> kubectl argo rollouts version
kubectl-argo-rollouts: v1.9.0+838d4e7
  BuildDate: 2026-03-20T21:11:48Z
  GitCommit: 838d4e792be666ec11bd0c80331e0c5511b5010e
  GitTreeState: clean
  GoVersion: go1.24.13
  Compiler: gc
  Platform: darwin/amd64
```



UI Dashboard

For the sake of completeness it needs to be mentioned that Argo Rollouts ships with a fully fledged UI Dashboard. It can be accessed via the kubectl argo rollouts dashboard command and provides a nice overview and basic commands for administration

```bash
> kubectl argo rollouts dashboard
INFO[0000] Argo Rollouts Dashboard is now available at http://localhost:3100/rollouts
```

http://localhost:3100/rollouts

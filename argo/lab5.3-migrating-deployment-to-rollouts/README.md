# 1. Prepare resources

```bash
kubectl create deploy nginx-deployment --image=nginx --replicas=3
```


```bash
> kubectl get po,deploy
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-5959b5b5c9-22gcl   1/1     Running   0          14s
pod/nginx-deployment-5959b5b5c9-fmttx   1/1     Running   0          14s
pod/nginx-deployment-5959b5b5c9-nm2m7   1/1     Running   0          14s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           14s
```


# 2. Convert Deployment to Rollout

```bash
kubectl apply -f convert-deployment-to-rollout.yaml
```

As a result, we have 6 nginx instances running, 3 managed by our vanilla deployment, 3 by the newly created rollout:

```bash
> kubectl get po,deploy
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-5959b5b5c9-22gcl   1/1     Running   0          4m8s
pod/nginx-deployment-5959b5b5c9-fmttx   1/1     Running   0          4m8s
pod/nginx-deployment-5959b5b5c9-nm2m7   1/1     Running   0          4m8s
pod/nginx-rollout-6d7df6cfcb-2clx6      1/1     Running   0          37s
pod/nginx-rollout-6d7df6cfcb-2nn7d      1/1     Running   0          37s
pod/nginx-rollout-6d7df6cfcb-d2kmq      1/1     Running   0          37s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   3/3     3            3           4m8s
```


# 3. Scale Down Deployment

```bash
kubectl scale deployment/nginx-deployment --replicas=0
```

```bash
> kubectl get po,deploy
NAME                                 READY   STATUS    RESTARTS   AGE
pod/nginx-rollout-6d7df6cfcb-2clx6   1/1     Running   0          105s
pod/nginx-rollout-6d7df6cfcb-2nn7d   1/1     Running   0          105s
pod/nginx-rollout-6d7df6cfcb-d2kmq   1/1     Running   0          105s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   0/0     0            0           5m16s
```

The scale down the Deployment can be done automatically via Argo Rollout controller. A special _scaledDown_ parameter exists:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: nginx-rollout
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  workloadRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
    scaleDown: onsuccess
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 10s}
```



```bash
# scalu up again
kubectl scale deployment/nginx-deployment --replicas=3
```


```bash
# migrate with scale down enabled
kubectl apply -f convert-deployment-to-rollout-with-scaledown.yaml
```

- it will create Rollout and scale down the Deployment


# 4. Clean Up Resources

```bash
kubectl delete rollout nginx-rollout
kubectl delete deployment nginx-deployment
```

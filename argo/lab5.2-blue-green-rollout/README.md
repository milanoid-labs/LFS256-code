# 1. Install Resources

```bash
# check existing rollouts
> kubectl get rollout
> No resources found in default namespace.
```


```bash
# create the Rollout resource
> kubectl apply -f rollout.yaml
rollout.argoproj.io/rollout-bluegreen created

# get the rollout
> kubectl get rollout
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rollout-bluegreen   2
```

- not ready yet


```bash
> kubectl describe rollouts.argoproj.io rollout-bluegreen
...
Status:
  Conditions:
    Last Transition Time:  2026-07-17T05:27:32Z
    Last Update Time:      2026-07-17T05:27:32Z
    Message:               The Rollout "rollout-bluegreen" is invalid: spec.strategy.blueGreen.activeService: Invalid value: "rollout-bluegreen-active": service "rollout-bluegreen-active" not found
    Reason:                InvalidSpec
    Status:                True
    Type:                  InvalidSpec
  Message:                 InvalidSpec: The Rollout "rollout-bluegreen" is invalid: spec.strategy.blueGreen.activeService: Invalid value: "rollout-bluegreen-active": service "rollout-bluegreen-active" not found
  Observed Generation:     1
  Phase:                   Degraded
Events:
  Type    Reason                  Age   From                 Message
  ----    ------                  ----  ----                 -------
  Normal  RolloutAddedToInformer  116s  rollouts-controller  Rollout resource added to informer: default/rollout-bluegreen
...
```

- we need to create the services the rollout refers to

```bash
kubectl apply -f service-blue-green-active.yaml -f service-blue-green-preview.yaml
```


Now the rollout is ready:

```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
Replicas:
  Desired:       2
  Current:       2
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE  INFO
⟳ rollout-bluegreen                            Rollout     ✔ Healthy  11m
└──# revision:1
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  25s  stable,active
      ├──□ rollout-bluegreen-5ffd47b8d4-cww4z  Pod         ✔ Running  15s  ready:1/1
      └──□ rollout-bluegreen-5ffd47b8d4-dtrm9  Pod         ✔ Running  15s  ready:1/1
```


# 2. Perform an Upgrade

- change blue to green

```bash
kubectl patch rollout rollout-bluegreen \
  --type=merge \
  -p='{"spec":{"template":{"spec":{"containers":[{"name":"rollouts-demo","image":"argoproj/rollouts-demo:green"}]}}}}'
```


- now there are 2 ReplicaSets

```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
                 argoproj/rollouts-demo:green (preview)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE    INFO
⟳ rollout-bluegreen                            Rollout     ॥ Paused   26m
├──# revision:3
│  └──⧉ rollout-bluegreen-84d95ffc99           ReplicaSet  ✔ Healthy  2m16s  preview
│     ├──□ rollout-bluegreen-84d95ffc99-9vmld  Pod         ✔ Running  2m16s  ready:1/1
│     └──□ rollout-bluegreen-84d95ffc99-nhlmz  Pod         ✔ Running  2m16s  ready:1/1
└──# revision:1
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  15m    stable,active
      ├──□ rollout-bluegreen-5ffd47b8d4-cww4z  Pod         ✔ Running  15m    ready:1/1
      └──□ rollout-bluegreen-5ffd47b8d4-dtrm9  Pod         ✔ Running  15m    ready:1/1
```

checks replicasets

```bash
> kubectl get replicaset
NAME                           DESIRED   CURRENT   READY   AGE
rollout-bluegreen-5ffd47b8d4   2         2         2       16m
rollout-bluegreen-84d95ffc99   2         2         2       3m11s
```


promote a new version

```bash
kubectl argo rollouts promote rollout-bluegreen
```

- now green is active/stable


```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue
                 argoproj/rollouts-demo:green (stable, active)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE    INFO
⟳ rollout-bluegreen                            Rollout     ✔ Healthy  27m
├──# revision:3
│  └──⧉ rollout-bluegreen-84d95ffc99           ReplicaSet  ✔ Healthy  3m52s  stable,active
│     ├──□ rollout-bluegreen-84d95ffc99-9vmld  Pod         ✔ Running  3m52s  ready:1/1
│     └──□ rollout-bluegreen-84d95ffc99-nhlmz  Pod         ✔ Running  3m52s  ready:1/1
└──# revision:1
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  17m    delay:20s
      ├──□ rollout-bluegreen-5ffd47b8d4-cww4z  Pod         ✔ Running  16m    ready:1/1
      └──□ rollout-bluegreen-5ffd47b8d4-dtrm9  Pod         ✔ Running  16m    ready:1/1
```


- the old blue Scales Down

```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:green (stable, active)
Replicas:
  Desired:       2
  Current:       2
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS        AGE  INFO
⟳ rollout-bluegreen                            Rollout     ✔ Healthy     28m
├──# revision:3
│  └──⧉ rollout-bluegreen-84d95ffc99           ReplicaSet  ✔ Healthy     5m   stable,active
│     ├──□ rollout-bluegreen-84d95ffc99-9vmld  Pod         ✔ Running     5m   ready:1/1
│     └──□ rollout-bluegreen-84d95ffc99-nhlmz  Pod         ✔ Running     5m   ready:1/1
└──# revision:1
   └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  • ScaledDown  18m
```


on the service note the hash of the new ReplicaSet


```bash
> kubectl describe svc rollout-bluegreen-active
Name:                     rollout-bluegreen-active
Namespace:                default
Labels:                   app=rollout-bluegreen-active
Annotations:              argo-rollouts.argoproj.io/managed-by-rollouts: rollout-bluegreen
Selector:                 app=rollout-bluegreen,rollouts-pod-template-hash=84d95ffc99
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.152.52
IPs:                      10.43.152.52
Port:                     80  80/TCP
TargetPort:               80/TCP
Endpoints:                10.42[118;1:3u.0.85:80,10.42.0.86:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```



# 3. Perform a Rollback

something happened and we want to rollback (revert) the promoted RS back from green to blue

```bash
# rollback
> kubectl argo rollouts undo rollout-bluegreen
rollout 'rollout-bluegreen' undo
```


```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (preview)
                 argoproj/rollouts-demo:green (stable, active)
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE    INFO
⟳ rollout-bluegreen                            Rollout     ॥ Paused   32m
├──# revision:4
│  └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  22m    preview
│     ├──□ rollout-bluegreen-5ffd47b8d4-hs4fz  Pod         ✔ Running  17s    ready:1/1
│     └──□ rollout-bluegreen-5ffd47b8d4-zg72s  Pod         ✔ Running  17s    ready:1/1
└──# revision:3
   └──⧉ rollout-bluegreen-84d95ffc99           ReplicaSet  ✔ Healthy  8m49s  stable,active
      ├──□ rollout-bluegreen-84d95ffc99-9vmld  Pod         ✔ Running  8m49s  ready:1/1
      └──□ rollout-bluegreen-84d95ffc99-nhlmz  Pod         ✔ Running  8m49s  ready:1/1
```


Note, that “undo” alone did not set the blue image active. The rollout is now again in the pausing phase, waiting for promotion of the rollout. Run the following command:


```bash
> kubectl argo rollouts promote rollout-bluegreen
rollout 'rollout-bluegreen' promoted
```


now it's set back to blue
```bash
> kubectl argo rollouts get ro rollout-bluegreen
Name:            rollout-bluegreen
Namespace:       default
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:blue (stable, active)
                 argoproj/rollouts-demo:green
Replicas:
  Desired:       2
  Current:       4
  Updated:       2
  Ready:         2
  Available:     2

NAME                                           KIND        STATUS     AGE    INFO
⟳ rollout-bluegreen                            Rollout     ✔ Healthy  35m
├──# revision:4
│  └──⧉ rollout-bluegreen-5ffd47b8d4           ReplicaSet  ✔ Healthy  25m    stable,active
│     ├──□ rollout-bluegreen-5ffd47b8d4-hs4fz  Pod         ✔ Running  3m23s  ready:1/1
│     └──□ rollout-bluegreen-5ffd47b8d4-zg72s  Pod         ✔ Running  3m23s  ready:1/1
└──# revision:3
   └──⧉ rollout-bluegreen-84d95ffc99           ReplicaSet  ✔ Healthy  11m    delay:15s
      ├──□ rollout-bluegreen-84d95ffc99-9vmld  Pod         ✔ Running  11m    ready:1/1
      └──□ rollout-bluegreen-84d95ffc99-nhlmz  Pod         ✔ Running  11m    ready:1/1
```

also see it on the service hash

```bash
> kubectl describe svc rollout-bluegreen-active
Name:                     rollout-bluegreen-active
Namespace:                default
Labels:                   app=rollout-bluegreen-active
Annotations:              argo-rollouts.argoproj.io/managed-by-rollouts: rollout-bluegreen
Selector:                 app=rollout-bluegreen,rollouts-pod-template-hash=5ffd47b8d4
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.152.52
IPs:                      10.43.152.52
Port:                     80  80/TCP
TargetPort:               80/TCP
Endpoints:                10.42.0.87:80,10.42.0.88:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```


# 4. cleanup resources


```bash
kubectl delete rollout rollout-bluegreen
kubectl delete svc rollout-bluegreen-active
```

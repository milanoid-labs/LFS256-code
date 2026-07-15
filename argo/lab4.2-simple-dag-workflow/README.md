# Lab 4.2. A Simple DAG Workflows

install resources

```bash
# tamplete
cat <<EOF > dag-workflow-template.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: echo-template
  namespace: argo
spec:
  serviceAccountName: argo
  templates:
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
EOF
```

```bash
cat <<EOF > dag-workflow.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond
  namespace: argo
spec:
  entrypoint: diamond
  templates:
    - name: diamond
      dag:
        tasks:
        - name: A
          arguments:
            parameters: [{name: message, value: A}]
          templateRef:
            name: echo-template
            template: echo
        - name: B
          dependencies: [A]
          templateRef:
            name: echo-template
            template: echo
          arguments:
            parameters: [{name: message, value: B}]
        - name: C
          dependencies: [A]
          templateRef:
            name: echo-template
            template: echo
          arguments:
            parameters: [{name: message, value: C}]
        - name: D
          dependencies: [B, C]
          templateRef:
            name: echo-template
            template: echo
          arguments:
            parameters: [{name: message, value: D}]
EOF
```

To successfully deploy this Workflow we need to temporarily grant admin permissions to argo ServiceAccount with the following command:

`kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=argo:default -n argo`

For a production environment, please follow the [official documentation](https://argo-workflows.readthedocs.io/en/latest/service-accounts/).

Create the WorkflowTemplate

```bash
kubectl apply -f dag-workflow-template.yaml
```

Submit Workflow

```bash
kubectl create -f dag-workflow.yaml
```


Inspect the created workflow

```bash
kubectl -n argo get workflowtemplates.argoproj.io
```


```bash
> kubectl -n argo get workflow
NAME               STATUS      AGE   MESSAGE
dag-diamondxhzxw   Succeeded   58s

> argo -n argo list
NAME               STATUS      AGE   DURATION   PRIORITY   MESSAGE
dag-diamondxhzxw   Succeeded   1m    40s        0
```

```bash
# get details
> argo -n argo get dag-diamondxhzxw
Name:                dag-diamondxhzxw
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Succeeded
Conditions:
 PodRunning          False
 Completed           True
Created:             Wed Jul 15 10:36:23 +0200 (1 minute ago)
Started:             Wed Jul 15 10:36:23 +0200 (1 minute ago)
Finished:            Wed Jul 15 10:37:03 +0200 (1 minute ago)
Duration:            40 seconds
Progress:            4/4
ResourcesDuration:   0s*(1 cpu),14s*(100Mi memory)

STEP                 TEMPLATE            PODNAME                           DURATION  MESSAGE
 ✔ dag-diamondxhzxw  diamond
 ├─✔ A               echo-template/echo  dag-diamondxhzxw-echo-2250723928  10s
 ├─✔ B               echo-template/echo  dag-diamondxhzxw-echo-2301056785  5s
 ├─✔ C               echo-template/echo  dag-diamondxhzxw-echo-2284279166  3s
 └─✔ D               echo-template/echo  dag-diamondxhzxw-echo-2334612023  3s
```

```bash
# read logs
> argo -n argo logs dag-diamondxhzxw
dag-diamondxhzxw-echo-2250723928: A
dag-diamondxhzxw-echo-2250723928: time="2026-07-15T08:36:33.384Z" level=info msg="sub-process exited" argo=true error="<nil>"
dag-diamondxhzxw-echo-2284279166: C
dag-diamondxhzxw-echo-2301056785: B
dag-diamondxhzxw-echo-2284279166: time="2026-07-15T08:36:46.157Z" level=info msg="sub-process exited" argo=true error="<nil>"
dag-diamondxhzxw-echo-2301056785: time="2026-07-15T08:36:47.087Z" level=info msg="sub-process exited" argo=true error="<nil>"
dag-diamondxhzxw-echo-2334612023: D
dag-diamondxhzxw-echo-2334612023: time="2026-07-15T08:36:55.154Z" level=info msg="sub-process exited" argo=true error="<nil>"
```

A run first, then B, C in parallel, then D


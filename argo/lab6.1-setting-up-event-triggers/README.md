
# 1. Prereq

- [x] Argo Workflows installed


# 2. Install Argo Events and the Needed components


```bash
kubectl create namespace argo-events
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml
```

The next command applies a validating webhook for Argo Events. Validating webhooks are used to ensure that incoming requests to the Kubernetes API server are valid:
```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install-validating-webhook.yaml
```



For setting up a native EventBus in the argo-events namespace, which handles event transportation in Argo Events, apply the configuration with this command:

```bash
kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```


Next we need to define an EventSource configuration that listens for webhook events in Argo Events, apply the following configuration using this command:

```bash
kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/event-sources/webhook.yaml
```


For the Sensor to properly interact with Kubernetes resources, apply the necessary RBAC policies with the command below:

```bash
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/sensor-rbac.yaml
```



Similarly, apply RBAC policies for Workflows to ensure they have the necessary permissions in Kubernetes. Run the following command:

```bash
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/rbac/workflow-rbac.yaml
```



Set up a Sensor to trigger workflows based on webhook events by creating this Sensor configuration in a file called webhook.yaml. Run the following command:

```yaml
# sensor.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: webhook-workflow-trigger
        k8s:
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook
              spec:
                entrypoint: cowsay
                arguments:
                  parameters:
                    - name: message
                      # the value will get overridden
                      #  by event payload from test-dep
                      value: hello world
                templates:
                  - name: cowsay
                    inputs:
                      parameters:
                        - name: message
                    container:
                      image: rancher/cowsay:latest
                      command: [cowsay]
                      args: ["{{inputs.parameters.message}}"]
          parameters:
            - src:
                dependencyName: test-dep
                dataKey: body
              dest: spec.arguments.parameters.0.value
```


```bash
kubectl -n argo-events apply -f sensor.yaml
```


```bash
# Expose the event-source pod via port forwarding to consume requests over HTTP:
kubectl -n argo-events port-forward $(kubectl -n argo-events get pod -l eventsource-name=webhook -o name) 12000:12000 &
```

Finally, simulate an external event that triggers the workflow. Send a test webhook event to the Event Source with this curl command:

```bash
curl -d '{"message":"this is my first webhook"}' -H "Content-Type: application/json" -X POST http://localhost:12000/example
```

Then in Argo Workflows UI running at http://capa-argo.milanoid.net/workflows/argo

![[Screenshot 2026-07-17 at 11.48.05.png]]


We have setup integration of Argo Workflows and Argo Events:

A webhook Event initiate an Argo Workflow showcasting the system's capability to reponse to events.


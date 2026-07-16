# Lab 4.3 CI/CD Using Argo Workflows

## 1. Create Argo Workflow

- [ ] modify to an actual CI/CD - maybe reuse Python app https://github.com/milanoid-labs/devops-study-app

Workflow steps:

- Build: The build step builds the image with the latest changes, using a Python 3 image for this scenario.
- Tests: The test step mounts a volume with test files and runs unit tests with the Python unittest library.
- Deployment: The deploy step runs the Python container and prints deploy. Normally, this step would involve pushing the tested code to a container registry, like AWS ECR or Harbor, and then deploying it to the production environment.

## 2. Deploy the workflow

```bash
kubectl -n argo apply -f workflow-ci.yaml
```

- this will create the resources and also starts the workflow

## 3. Inspect the workflow

```bash
> argo -n argo get python-app
Name:                python-app
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Succeeded
Conditions:
 PodRunning          False
 Completed           True
Created:             Thu Jul 16 14:51:07 +0200 (2 minutes ago)
Started:             Thu Jul 16 14:51:07 +0200 (2 minutes ago)
Finished:            Thu Jul 16 14:52:01 +0200 (1 minute ago)
Duration:            54 seconds
Progress:            3/3
ResourcesDuration:   2s*(1 cpu),31s*(100Mi memory)

STEP           TEMPLATE    PODNAME                      DURATION  MESSAGE
 ✔ python-app  python-app
 ├───✔ build   build       python-app-build-3831382643  25s
 ├───✔ test    test        python-app-test-461837862    3s
 └───✔ deploy  deploy      python-app-deploy-988129288  3s
```


```bash
# logs
> argo -n argo logs python-app
python-app-build-3831382643: build
python-app-build-3831382643: time="2026-07-16T12:51:31.314Z" level=info msg="sub-process exited" argo=true error="<nil>"
python-app-test-461837862:
python-app-test-461837862: ----------------------------------------------------------------------
python-app-test-461837862: Ran 0 tests in 0.000s
python-app-test-461837862:
python-app-test-461837862: OK
python-app-test-461837862: time="2026-07-16T12:51:43.605Z" level=info msg="sub-process exited" argo=true error="<nil>"
python-app-deploy-988129288: deploy
python-app-deploy-988129288: time="2026-07-16T12:51:53.652Z" level=info msg="sub-process exited" argo=true error="<nil>"
```


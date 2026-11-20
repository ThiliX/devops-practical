# DevOps Practical

A small Python Flask web app taken through a DevOps lifecycle: version control, continuous integration, containerisation, orchestration with Kubernetes and a chaos (self-healing) test.

- **Repository:** https://github.com/ThiliX/devops-practical
- **Docker Hub image:** [`thilix7/flask-app:v1`](https://hub.docker.com/r/thilix7/flask-app)

## Status

| Phase | DevOps concept | Status |
|---|---|---|
| 1. Foundation | Version control & "Everything as Code" | ✅ Done |
| 2. Continuous Integration | Shift-left testing & automation | ✅ Done |
| 3. Containerisation | Immutable infrastructure & environment parity | ✅ Done |
| 4. Orchestration | High availability, desired state & self-healing | ✅ Done |
| 5. Chaos Test | Resilience | ✅ Done |
| 6. Continuous Deployment | CD & GitOps | ❌ Not done ([see below](#phase-6--continuous-deployment-not-done)) |

## Project structure

```
devops-practical/
├── .github/workflows/ci.yml   # GitHub Actions CI pipeline
├── k8s/
│   ├── deployment.yaml        # Kubernetes Deployment (2 replicas)
│   └── service.yaml           # Kubernetes NodePort Service
├── app.py                     # Flask application
├── test_app.py                # Automated test (quality gate)
├── requirements.txt           # Dependencies
├── Dockerfile                 # Container image definition
├── .dockerignore
└── .gitignore
```

## Prerequisites

- Git
- Python 3.12 (only needed to run the app or tests without Docker)
- Docker Desktop
- kubectl
- Minikube

---

## Phase 1 – The Foundation (Source Control)

**Goal:** create the app, write a test, and push it to a central repository.

1. Create the project folder and initialise Git:
   ```bash
   mkdir devops-practical
   cd devops-practical
   git init
   ```
2. Add the dependencies in `requirements.txt`:
   ```
   flask==3.0.0
   pytest==7.4.0
   ```
3. Write the app in `app.py`. It reads the `PORT` environment variable (for cloud hosts) and falls back to 5000 locally:
   ```python
   import os
   from flask import Flask

   app = Flask(__name__)

   @app.route('/')
   def home():
       return "CI/CD and Docker Pipeline is a massive success! V2"

   if __name__ == '__main__':
       port = int(os.environ.get('PORT', 5000))
       app.run(host='0.0.0.0', port=port)
   ```
4. Write the automated test in `test_app.py`:
   ```python
   from app import app

   def test_home():
       client = app.test_client()
       response = client.get('/')
       assert response.status_code == 200
       assert b"CI/CD and Docker Pipeline is a massive success! V2" in response.data
   ```
5. Create an empty public repository named `devops-practical` on GitHub, then commit and push:
   ```bash
   git add .
   git commit -m "Initial commit: App and tests"
   git branch -M main
   git remote add origin https://github.com/ThiliX/devops-practical.git
   git push -u origin main
   ```

## Phase 2 – Continuous Integration (GitHub Actions)

**Goal:** run the tests automatically on every push, so no one has to check the code by hand.

1. Create the workflow folder:
   ```bash
   mkdir -p .github/workflows
   ```
2. Add `.github/workflows/ci.yml`:
   ```yaml
   name: CI Pipeline
   on:
     push:
       branches: [ "main" ]
   jobs:
     build-and-test:
       runs-on: ubuntu-latest
       steps:
         - name: Checkout Code
           uses: actions/checkout@v5
         - name: Set up Python
           uses: actions/setup-python@v6
           with:
             python-version: '3.12'
         - name: Install Dependencies
           run: |
             pip install -r requirements.txt
         - name: Run PyTest
           run: pytest
   ```
3. Commit and push. The pipeline runs automatically; check the result in the **Actions** tab:
   ```bash
   git add .github/
   git commit -m "Add CI pipeline"
   git push
   ```

> **Change from the guide:** the guide uses Python 3.9 with `actions/checkout@v4` and `actions/setup-python@v4`. The first run failed at "Set up Python" because GitHub's `ubuntu-latest` runner (Ubuntu 26.04) no longer provides Python 3.9. The workflow and the Dockerfile were both moved to Python 3.12, so CI and the container use the same version, and the actions were updated to their current versions. The next run [passed](https://github.com/ThiliX/devops-practical/actions/runs/44496326245).

## Phase 3 – Containerisation (Docker)

**Goal:** package the app and its environment into a standard unit.

1. Add the `Dockerfile`:
   ```dockerfile
   FROM python:3.12-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   EXPOSE 5000
   CMD ["python", "app.py"]
   ```
2. Build the image and run it:
   ```bash
   docker build -t flask-app:v1 .
   docker run -p 5000:5000 flask-app:v1
   ```
   Open http://localhost:5000, then stop the container with `Ctrl+C`.
3. Optionally, run the tests inside the container:
   ```bash
   docker run --rm flask-app:v1 pytest -q
   ```
4. Push the image to Docker Hub so Kubernetes can pull it:
   ```bash
   docker login
   docker tag flask-app:v1 thilix7/flask-app:v1
   docker push thilix7/flask-app:v1
   ```

**Result:** the test passed inside the container, http://localhost:5000 showed the app's message, and the image is published as `thilix7/flask-app:v1`.

## Phase 4 – Orchestration (Kubernetes on Minikube)

**Goal:** deploy the container to a local Kubernetes cluster.

1. Install the tools and check them (Windows shown; on Mac use `brew install kubectl minikube`):
   ```bash
   winget install Kubernetes.minikube
   kubectl version --client
   minikube version
   ```
2. Start the cluster:
   ```bash
   minikube start --driver=docker
   ```
3. The manifests are in `k8s/`. `deployment.yaml` keeps 2 replicas of the container running:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: flask-deployment
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: flask-web
     template:
       metadata:
         labels:
           app: flask-web
       spec:
         containers:
         - name: flask-container
           image: thilix7/flask-app:v1
           ports:
           - containerPort: 5000
   ```
   `service.yaml` exposes the pods on port 80:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-service
   spec:
     selector:
       app: flask-web
     ports:
       - protocol: TCP
         port: 80
         targetPort: 5000
     type: NodePort
   ```
4. Deploy and watch the pods start:
   ```bash
   kubectl apply -f k8s/
   kubectl get pods -w
   ```
5. Open the app in a browser:
   ```bash
   minikube service flask-service
   ```

**Result:** Minikube pulled `thilix7/flask-app:v1` from Docker Hub and both pods were running:

```
NAME                                READY   STATUS    RESTARTS   AGE
flask-deployment-778cd7f577-24ws2   1/1     Running   0          26s
flask-deployment-778cd7f577-cxgdd   1/1     Running   0          32s
```

## Phase 5 – Chaos Test (Self-Healing)

**Goal:** show that Kubernetes repairs failures automatically.

1. Get the name of a running pod:
   ```bash
   kubectl get pods
   ```
2. Delete it to simulate a crash:
   ```bash
   kubectl delete pod <POD-NAME>
   ```
3. Check the pods again straight away:
   ```bash
   kubectl get pods
   ```

**Result:** Kubernetes created a replacement immediately, and it was running within about 9 seconds, keeping the desired 2 replicas:

```
$ kubectl delete pod flask-deployment-65f964dd4d-6fhhg

$ kubectl get pods            # immediately after
NAME                                READY   STATUS              RESTARTS   AGE
flask-deployment-65f964dd4d-6fhhg   1/1     Terminating         0          42s
flask-deployment-65f964dd4d-hv8rl   0/1     ContainerCreating   0          0s
flask-deployment-65f964dd4d-n6w8t   1/1     Running             0          42s

$ kubectl get pods            # 8 seconds later
NAME                                READY   STATUS        RESTARTS   AGE
flask-deployment-65f964dd4d-6fhhg   1/1     Terminating   0          51s
flask-deployment-65f964dd4d-hv8rl   1/1     Running       0          9s
flask-deployment-65f964dd4d-n6w8t   1/1     Running       0          51s
```

## Phase 6 – Continuous Deployment (not done)

The guide's last phase deploys the app to Render, so that every `git push` tests, builds and publishes it to a public URL with no manual steps.

**This phase was not completed.** What was done:

- `app.py` was prepared for cloud hosting: it reads the `PORT` the host assigns and falls back to 5000 locally.
- The `Dockerfile` is ready for a host that builds from it.

Why it could not be finished:

- **Render** requires a card to verify the account, and the card was declined by the bank.
- **Hugging Face Spaces** now limits Docker Spaces to paid plans.
- **Back4app Containers** returned an error during sign-up.

So there is no public live URL, and the "no-touch" deployment test (change the message, push, see the live site update) was not demonstrated.

### Steps to finish it later

1. On a host that builds from a Dockerfile (for example Render), create a web service from this GitHub repository.
2. Use branch `main`, runtime **Docker** and the free instance type.
3. Wait for the first deploy and open the public URL.
4. Change the message in `app.py` and the matching text in `test_app.py`, then commit and push. CI tests the change, and the host redeploys it automatically.

---

## Useful commands

| Task | Command |
|---|---|
| Run tests locally | `pip install -r requirements.txt` then `pytest` |
| Run the app locally | `python app.py`, then open http://localhost:5000 |
| Start / stop the cluster | `minikube start` / `minikube stop` |
| See the pods | `kubectl get pods` |
| Open the app on Kubernetes | `minikube service flask-service` |
| Remove the deployment | `kubectl delete -f k8s/` |

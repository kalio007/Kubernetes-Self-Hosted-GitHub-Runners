
# GitHub Actions Self-Hosted Runner Deployment on Kubernetes

This repository contains the configuration and setup instructions for deploying a self-hosted GitHub Actions runner on a Kubernetes cluster using the Actions Runner Controller.

## Overview

The `runner.yaml` manifest provided in this repository defines a Kubernetes RunnerDeployment, which deploys a self-hosted GitHub Actions runner in your Kubernetes cluster. This runner is configured to work with the `prime-kube/hng_boilerplate_golang_web` repository and is labeled as `Kalio`.

## Prerequisites

Before deploying the runner, ensure you have the following prerequisites:

- **Kubernetes Cluster**: A running Kubernetes cluster. This guide uses Minikube to set up a local Kubernetes cluster.

### Docker Installation

Minikube requires Docker to manage the containers. Install Docker with the following commands:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
```

### Minikube Installation

Install Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Start the Minikube cluster:

```bash
minikube start --driver=docker
```

### kubectl Installation

`kubectl` is the command-line tool for interacting with your Kubernetes cluster. Install `kubectl` by following the [official documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

### Helm Installation

Helm is the package manager for Kubernetes. Install Helm using the following commands:

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Generate a Personal Access Token (PAT)

To authenticate the Actions Runner Controller with GitHub, you need a Personal Access Token (PAT) with the following scopes:

1. Go to [GitHub Settings > Developer settings > Personal access tokens > Tokens (classic)](https://github.com/settings/tokens).
2. Click on "Generate new token."
3. Select the following scopes:
   - `repo`: Full control of private repositories
   - `admin:org`: Read and write access to organization resources
4. Click "Generate token" and copy the token. You'll need this when installing the Actions Runner Controller.

## Deployment Instructions

### 1. Install Cert-Manager

Cert-Manager is required for securing communication within the Kubernetes cluster.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

### 2. Deploy the Actions Runner Controller

Use Helm to install the Actions Runner Controller on your Kubernetes cluster.

```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm upgrade --install --namespace actions-runner-system --create-namespace \
             --set=authSecret.create=true \
             --set=authSecret.github_token="YOUR_GITHUB_TOKEN" \
             --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

Replace `YOUR_GITHUB_TOKEN` with the Personal Access Token (PAT) you generated earlier.

### 3. Deploy the GitHub Actions Runner

Apply the `runner.yaml` manifest to deploy the runner in your Kubernetes cluster:

```bash
kubectl apply -f runner.yaml
```

### 4. Verify the Deployment

Check if the runner pod is running:

```bash
kubectl get pods -w
```

You should see a pod named `kalio-runner` in the Running state.

### 5. Run the Logs Command

Now, run the `kubectl logs` command to view the logs for the pod:

```bash
kubectl logs -f <POD_NAME>
```

## runner.yaml Configuration

The `runner.yaml` file is configured as follows:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: kalio-runner
spec:
  replicas: 1
  template:
    spec:
      repository: prime-kube/hng_boilerplate_golang_web
      labels:
      - Kalio
```

- **name**: The name of the runner deployment.
- **replicas**: Number of runner instances.
- **repository**: The GitHub repository this runner will be associated with.
- **labels**: Labels used to identify and organize the runner.

## Workflow Configuration

To use this self-hosted runner in your GitHub Actions workflows, specify the runner label in your workflow file:

```yaml
name: Build Binary and Run Tests

on: workflow_dispatch

jobs:
  build_and_test:
    runs-on: Kalio    
    env:
      TEST_USERNAME: postgres
      TEST_PASSWORD: password
      TEST_DB_NAME: db_name
      TEST_DB_HOST: localhost
      TEST_DB_PORT: 5432
      TEST_DB_CONNECTION: pgsql
      TEST_TIMEZONE: Africa/Lagos
      TEST_SSLMODE: disable
      TEST_MIGRATE: true
      REDIS_PORT: 6379
      REDIS_HOST: localhost
      REDIS_DB: 0

    services:
      postgres:
        image: postgres:latest
        env:
          POSTGRES_USER: ${{ env.TEST_USERNAME }}
          POSTGRES_PASSWORD: ${{ env.TEST_PASSWORD }}
          POSTGRES_DB: ${{ env.TEST_DB_NAME }}
        ports:
          - 5432:5432
      redis:
        image: redis
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      - name: Set Golang
        uses: actions/setup-go@v4
        with:
          go-version: "1.22.1"
      - name: Create app config file
        run: cp app-sample.env app.env
      - name: Build the application
        run: go build -o development_app
      - name: Run the application
        run: nohup ./development_app > /dev/null 2>&1 &
      - name: Wait for application to start
        run: |
          for i in {1..30}; do
            curl -s http://localhost:8019 && break
            sleep 1
          done
      - name: Run All Tests
        run: go test ./... -timeout 300s
```

In the `runs-on` field, specify the label `Kalio` to ensure the job runs on your self-hosted runner.
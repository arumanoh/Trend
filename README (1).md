# Trend App — DevOps Assessment

Deployment of the Trend (Trendify) React application through a full AWS production pipeline: Docker, Terraform, EKS, Jenkins CI/CD, and Prometheus/Grafana monitoring.

## Architecture Overview

- **Application**: Pre-built React (Vite) SPA served via nginx on port 3000
- **Containerization**: Docker image hosted on DockerHub (`arumanoh/trend-app`)
- **Infrastructure**: Provisioned via Terraform (VPC, IAM, EC2 with Jenkins)
- **Orchestration**: AWS EKS cluster running the app via Deployment + LoadBalancer Service
- **CI/CD**: Jenkins declarative pipeline, triggered by a GitHub webhook on every push
- **Monitoring**: Prometheus + Grafana (`kube-prometheus-stack` via Helm) on EKS

## Repository Structure

```
.
├── dist/                  # Pre-built Vite app output
├── Dockerfile             # nginx-based image, serves on port 3000
├── nginx.conf             # SPA fallback routing config
├── .dockerignore
├── .gitignore
├── Jenkinsfile            # Declarative pipeline: build → push → deploy
├── k8s/
│   ├── deployment.yaml    # 2 replicas, resource limits, probes
│   └── service.yaml       # LoadBalancer (NLB) service
└── README.md
```

Step-by-step screenshots and a detailed walkthrough of the full setup are included in the accompanying documentation file in this repository.

## Setup Instructions

### 1. Docker

Build and run the containerized app locally or on an EC2 instance:

```bash
docker build -t trend-app:latest .
docker run -d -p 3000:3000 --name trend-app trend-app:latest
```

### 2. Infrastructure (Terraform)

Provisions a VPC, subnet, Internet Gateway, route table, security group, IAM role + instance profile, and an EC2 instance with Jenkins auto-installed via user-data (Java 21, Jenkins, Docker, kubectl, AWS CLI, eksctl).

```bash
cd trend-infra
terraform init
terraform plan
terraform apply
```

Outputs the new Jenkins instance's public IP and URL (`http://<ip>:8080`).

### 3. EKS Cluster

```bash
eksctl create cluster \
  --name trend-cluster \
  --region eu-north-1 \
  --nodegroup-name trend-nodes \
  --node-type t3.small \
  --nodes 2 --nodes-min 2 --nodes-max 3 \
  --managed
```

Map the Jenkins EC2's IAM role into the cluster's RBAC so Jenkins can deploy to it:

```bash
eksctl create iamidentitymapping \
  --cluster trend-cluster \
  --region eu-north-1 \
  --arn arn:aws:iam::<account-id>:role/trend-jenkins-ec2-role \
  --group system:masters \
  --username jenkins-deployer
```

### 4. Deploy to Kubernetes

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### 5. Jenkins Pipeline

- Plugins installed: Git, Pipeline, Docker Pipeline, Kubernetes, Kubernetes CLI (in addition to the suggested plugin set)
- Credentials configured in Jenkins: DockerHub (username/password) and GitHub (Personal Access Token)
- Pipeline job created as **Pipeline script from SCM**, pointed at this repository's `Jenkinsfile`, branch `main`
- GitHub webhook configured under **Settings → Webhooks** on this repo, pointing at `http://<jenkins-ip>:8080/github-webhook/`, triggering on push events

## Pipeline Explanation

The `Jenkinsfile` defines four stages, run on every push to `main`:

1. **Checkout** — pulls the latest commit from GitHub
2. **Build Docker Image** — builds the image, tagged with both the current Jenkins build number and `latest`
3. **Push to DockerHub** — authenticates using stored Jenkins credentials and pushes both tags
4. **Deploy to EKS** — applies the Kubernetes manifests, updates the running Deployment's image via `kubectl set image`, and waits for the rollout to complete successfully before finishing

Because the pipeline is triggered by the GitHub webhook, a `git push` to `main` is enough to kick off a fully automated build → push → deploy cycle with no manual intervention.

## Monitoring

Prometheus and Grafana are deployed via the `kube-prometheus-stack` Helm chart into a dedicated `monitoring` namespace on the EKS cluster, providing cluster-wide and pod-level metrics (CPU, memory, node health, network I/O, pod status) through pre-built Grafana dashboards.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```

Grafana is exposed via a LoadBalancer service for external access:

```bash
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

## Kubernetes LoadBalancer ARN

```
arn:aws:elasticloadbalancing:eu-north-1:367061003285:loadbalancer/net/abaa89aa3e8724ab6ab038e148f36329/12e99f13afd85626
```

## Documentation

See the accompanying documentation file in this repository for the complete step-by-step walkthrough with screenshots covering EC2 provisioning, Docker build/run, Terraform apply, Jenkins installation and pipeline configuration, EKS cluster/node status, application access via the LoadBalancer, pipeline run logs, and Grafana dashboards.

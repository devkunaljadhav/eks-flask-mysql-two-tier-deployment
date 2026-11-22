# Two-Tier Flask & MySQL Application on Amazon EKS

[![AWS](https://img.shields.io/badge/AWS-EKS-orange?logo=amazonaws)](https://aws.amazon.com/eks/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Manifests-blue?logo=kubernetes)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Python](https://img.shields.io/badge/Python-3.9-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-2.0.1-000000?logo=flask)](https://flask.palletsprojects.com/)

A complete two-tier application demonstrating how to containerize a Python/Flask web application and deploy it with MySQL on Amazon Elastic Kubernetes Service (EKS).

The Flask frontend accepts comments through a browser, stores them in a MySQL database, and displays the saved comments on the page.

---

## Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Application Features](#application-features)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
  - [Quick Deployment](#option-1-quick-deployment)
  - [Build and Deploy Your Own Image](#option-2-build-and-deploy-your-own-image)
- [Detailed EKS Deployment Guide](./Steps%20to%20Deploy%20Application%20in%20EKS.txt)
- [Run Locally with Docker](#run-locally-with-docker)
- [Database Initialization](#database-initialization)
- [Environment Variables](#environment-variables)
- [Health and Troubleshooting](#health-and-troubleshooting)
- [Screenshots](#screenshots)
- [Cleanup](#cleanup)
- [Production Improvements](#production-improvements)
- [Author](#author)
- [License](#license)

## Project Overview

| Component | Technology |
|---|---|
| Frontend and API | Python 3.9, Flask |
| Database | MySQL |
| Containerization | Docker |
| Container orchestration | Amazon EKS, Kubernetes |
| Cluster provisioning | eksctl |
| External access | Kubernetes LoadBalancer Service |
| Supporting resources | ConfigMap and Kubernetes Secret |

## Architecture

```text
Internet
   |
   | HTTP
   v
AWS Load Balancer
   |
   v
two-tier-app-service (LoadBalancer)
   |
   v
Flask pods (two-tier-app)
   |
   | Kubernetes Service DNS: mysql
   v
MySQL Service
   |
   v
MySQL pod
   |
   v
Container-local MySQL data
```

![Amazon EKS cross-VPC architecture](<./Amazon EKS Cross-VPC Architecture.png>)

## Repository Structure

```text
.
├── app.py                              # Flask application and database operations
├── Dockerfile                          # Python application container image
├── requirements.txt                    # Python dependencies
├── message.sql                         # Standalone MySQL table definition
├── templates/
│   └── index.html                      # Web interface
├── static/
│   └── sir.jpg                         # Static profile image
├── eks-manifests/
│   ├── mysql-secrets.yml               # MySQL credentials
│   ├── mysql-configmap.yml             # Database initialization script
│   ├── mysql-deployment.yml            # MySQL Deployment
│   ├── mysql-svc.yml                   # Internal MySQL Service
│   ├── two-tier-app-deployment.yml     # Flask Deployment
│   └── two-tier-app-svc.yml            # Public LoadBalancer Service
├── Screenshoots/
│   ├── cluster.png
│   ├── ec2-instances.png
│   └── github-repo.png
└── Steps to Deploy Application in EKS.txt
```

## Application Features

- Flask-based web interface
- Form submission using AJAX
- Automatic MySQL `messages` table initialization
- Kubernetes environment-based database configuration
- Internal database discovery through the `mysql` ClusterIP Service
- Public application access through an AWS Load Balancer
- Dockerized deployment with configurable container replicas

## Prerequisites

Before deploying, make sure you have:

- An AWS account with permission to create EKS clusters and related networking resources
- AWS CLI configured with valid credentials
- `kubectl`
- `eksctl`
- Docker
- Git
- A Docker Hub account if you plan to publish your own application image

Verify the command-line tools:

```bash
aws --version
eksctl version
kubectl version --client
docker --version
git --version
```

---

## Deployment

Follow either deployment option below. For a step-by-step walkthrough that includes installing the tools on an Amazon Linux host, see the [detailed EKS deployment guide](./Steps%20to%20Deploy%20Application%20in%20EKS.txt).

## Option 1: Quick Deployment

The included Flask Deployment currently references a prebuilt Docker image. After creating the cluster, the quickest deployment path is to apply the manifests directly.

### 1. Create the EKS Cluster

```bash
eksctl create cluster \
  --name kunal-cluster \
  --region ap-south-1 \
  --node-type t3.small \
  --nodes-min 2 \
  --nodes-max 3
```

Configure `kubectl` for the cluster:

```bash
aws eks update-kubeconfig \
  --name kunal-cluster \
  --region ap-south-1
```

Confirm that worker nodes are available:

```bash
kubectl get nodes
```

### 2. Deploy Kubernetes Resources

Apply the MySQL resources first, followed by the Flask application:

```bash
cd eks-manifests

kubectl apply -f mysql-secrets.yml
kubectl apply -f mysql-configmap.yml
kubectl apply -f mysql-deployment.yml
kubectl apply -f mysql-svc.yml

kubectl apply -f two-tier-app-deployment.yml
kubectl apply -f two-tier-app-svc.yml
```

### 3. Check the Deployment

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all
```

Wait until the `two-tier-app-service` has an external hostname:

```bash
kubectl get service two-tier-app-service --watch
```

Open the displayed external hostname or AWS Load Balancer address in a browser.

---

## Option 2: Build and Deploy Your Own Image

Use this option to change the application source or publish a customized container image.

### 1. Build the Image

From the repository root:

```bash
docker build -t <dockerhub-user>/eks-flask-mysql-project:latest .
```

### 2. Login and Push

```bash
docker login
docker push <dockerhub-user>/eks-flask-mysql-project:latest
```

Replace `image: reyadocker/eks-flask-mysql-project:latest` in `eks-manifests/two-tier-app-deployment.yml` with your image:

```yaml
image: <dockerhub-user>/eks-flask-mysql-project:latest
imagePullPolicy: Always
```

Apply the updated Deployment:

```bash
kubectl apply -f eks-manifests/two-tier-app-deployment.yml
kubectl rollout status deployment/two-tier-app
```

---

## Run Locally with Docker

The application requires a reachable MySQL database.

```bash
docker run -d \
  --name mysql \
  -e MYSQL_ROOT_PASSWORD=admin \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_USER=admin \
  -e MYSQL_PASSWORD=admin \
  -p 3306:3306 \
  mysql:latest
```

For Linux, find the Docker bridge gateway and use it as `MYSQL_HOST`. On Docker Desktop, `host.docker.internal` may be used instead.

```bash
docker build -t eks-flask-mysql-project .

docker run --rm -p 5000:5000 \
  -e MYSQL_HOST=<mysql-host> \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=admin \
  -e MYSQL_DB=mydb \
  eks-flask-mysql-project
```

Open `http://localhost:5000`.

> **Security notice:** The current manifests contain development-only database credentials and a public Flask debug server. Do not expose this configuration in production. Use Kubernetes Secrets, external secret management, a private database service, non-root database users, persistent storage, HTTPS, and a production WSGI server.

---

## Database Initialization

The `mysql-initdb-config` ConfigMap creates the database and `messages` table when MySQL initializes:

```sql
CREATE DATABASE IF NOT EXISTS mydb;
USE mydb;
CREATE TABLE messages (
  id INT AUTO_INCREMENT PRIMARY KEY,
  message TEXT
);
```

The Flask application also executes `CREATE TABLE IF NOT EXISTS messages` before starting.

## Environment Variables

The Flask container reads these environment variables:

| Variable | Kubernetes value | Purpose |
|---|---|---|
| `MYSQL_HOST` | `mysql` | Kubernetes DNS name of the MySQL Service |
| `MYSQL_USER` | `root` | MySQL username |
| `MYSQL_PASSWORD` | `admin` | MySQL password |
| `MYSQL_DB` | `mydb` | Database name |

For local execution, the application defaults to `localhost`, `default_user`, `default_password`, and `default_db` when variables are not supplied.

## Health and Troubleshooting

### Inspect Pods

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Inspect Services and Endpoints

```bash
kubectl get svc
kubectl get endpoints
```

### Restart the Flask Deployment

```bash
kubectl rollout restart deployment/two-tier-app
kubectl rollout status deployment/two-tier-app
```

### Check a Specific Application Pod

```bash
kubectl exec -it deployment/two-tier-app -- /bin/bash
```

### Common Issues

- **MySQL connection refused:** verify that the MySQL pod is running and that `MYSQL_HOST` resolves to the `mysql` Service.
- **Authentication failed:** confirm that the Flask `MYSQL_USER`, `MYSQL_PASSWORD`, and `MYSQL_DB` values match the MySQL Deployment.
- **LoadBalancer stays pending:** verify AWS permissions, subnet configuration, and AWS Load Balancer status.
- **Image not updating:** set a new image tag or run `kubectl rollout restart deployment/two-tier-app`.
- **Application restarts:** inspect pod logs with `kubectl logs <pod-name> --previous`.

---

## Screenshots

### EKS Cluster

![EKS cluster](./Screenshoots/cluster.png)

### EC2 Worker Instances

![EC2 instances](./Screenshoots/ec2-instances.png)

### Source Repository

![GitHub repository](./Screenshoots/github-repo.png)

## Cleanup

Delete the application resources first:

```bash
kubectl delete -f eks-manifests/two-tier-app-svc.yml
kubectl delete -f eks-manifests/two-tier-app-deployment.yml
```

Delete the MySQL resources and then the cluster:

```bash
kubectl delete -f eks-manifests/mysql-svc.yml
kubectl delete -f eks-manifests/mysql-deployment.yml
kubectl delete -f eks-manifests/mysql-configmap.yml
kubectl delete -f eks-manifests/mysql-secrets.yml

eksctl delete cluster \
  --name kunal-cluster \
  --region ap-south-1
```

Also verify in the AWS Console that no unneeded load balancers, volumes, or other cluster resources remain.

## Production Improvements

This repository is ideal as a DevOps learning project, but a production deployment should add:

- Kubernetes Secrets or AWS Secrets Manager for credentials
- AWS RDS for managed MySQL instead of a pod-based database
- Persistent volume storage or a managed database backup strategy
- Liveness and readiness probes
- Resource requests and limits
- Horizontal Pod Autoscaling
- Ingress with an ACM certificate and HTTPS
- Network Policies
- CI/CD with automated image builds and deployments
- Helm or Kustomize for environment configuration
- Health endpoints and centralized logging
- Container image vulnerability scanning
- A non-debug production Flask server

## Author

**Kunal Jadhav**

- 🌐 **Portfolio:** [kunalcloud.shop](https://kunalcloud.shop/)
- 🐙 **GitHub:** [@devkunaljadhav](https://github.com/devkunaljadhav)
- 💼 **LinkedIn:** [Kunal Jadhav](https://www.linkedin.com/in/devkunaljadhav)
- 📧 **Email:** [kunaljadhav1625@gmail.com](mailto:kunaljadhav1625@gmail.com)

## License

No license is currently specified. Add a license before reuse or redistribution.

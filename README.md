# Movie Picture Pipeline Project

This repository contains a microservices application (Frontend and Backend) deployed on an AWS managed Kubernetes cluster using Infrastructure as Code (IaC) and automated CI/CD pipelines.

## Infrastructure Architecture

The cloud infrastructure has been optimized for stability and high availability:

* **Container Orchestration:** Deployed on **Amazon EKS** (Elastic Kubernetes Service).
* **Infrastructure as Code (IaC):** Configured via Terraform. The `aws_eks_node_group` has been updated to support dynamic scaling from **3 to 5 nodes** (starting with a baseline of 3 nodes) to prevent resource starvation during deployments.
* **Deployment Strategy:** Automated via **Kustomize** for dynamic Docker image tag injection from **Amazon ECR**.

---

## CI/CD Master Workflow

The deployment pipeline is fully automated and orchestrated through a single unified workflow file (`master-pipeline.yaml`) using GitHub Actions. It enforces a strict sequential execution by inheriting repository secrets (`secrets: inherit`):

[backend-ci] ──> [backend-cd] ──> [frontend-ci] ──> [frontend-cd]

1. **`backend-ci`:** Lints, tests, and builds the backend Docker image.
2. **`backend-cd`:** Safely deletes old backend pods via `kubectl delete` to free cluster capacity, then applies new manifests using Kustomize.
3. **`frontend-ci`:** Lints, tests, and builds the frontend web client.
4. **`frontend-cd`:** Deploys the frontend client to the EKS cluster.

---

## Deployment Guide

### Prerequisites
* Active AWS CLI credentials (federated or admin profile).
* Terraform and AWS CLI installed locally.

### Step 1: Provision Infrastructure
Navigate to your Terraform directory and run:
```bash
terraform init
terraform apply -auto-approve
```

### Step 2: Configure GitHub Secrets for 'github-action-user' IAM user.
Add the following credentials under GitHub **Repository Secrets**:
* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`

### Step 3: Add GitHub Action user to Kubernetes.
Add the github-action-user IAM user ARN to the Kubernetes configuration to allow that user to execute kubectl commands against the cluster. To do this, run the init.sh helper script in the setup folder.

Note: The commands below assume the cluster name is cluster, which is the name used in the Terraform settings. If you used a different name for your cluster, adjust the --name parameter of the first command accordingly.

aws eks update-kubeconfig --name cluster --region us-east-1
kubectl get configmap aws-auth -n kube-system
cd /workspace/setup
./init.sh

### Step 4: Run the Pipeline
Push a commit to the `main` branch, or manually trigger the `Master Workflow` from the **Actions** tab in GitHub.

---

## Deployment Evidences (Requirement 5)

The entire system is verified and running under the following public endpoints:

### 1. Pipeline Status
* **Status:** Success (All jobs passed).
* *Artifact reference: See attached `pipeline-success.png`*

### 2. Backend API Verification
* **Endpoint:** `http://a674f6d8a8f7c4467829bf13611b886f-557511934.us-east-1.elb.amazonaws.com/movies`
* **Response Status:** `HTTP/1.1 200 OK`
* *Artifact reference: See attached `backend-working.png` (Terminal JSON response)*

### 3. Frontend UI Verification
* **Endpoint:** `http://aee6ae31e950d43da8e517d7b18dc8ec-823855176.us-east-1.elb.amazonaws.com/`
* **Behavior:** Successfully renders the UI and fetches live data from the backend API.
* *Artifact reference: See attached `frontend-working.png` (Web Browser view)*

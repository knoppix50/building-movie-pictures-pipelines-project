# Movie Picture Pipeline Project

This repository contains a microservices application (Frontend and Backend) deployed on an AWS managed Kubernetes cluster using Infrastructure as Code (IaC) and automated CI/CD pipelines.

## Infrastructure Architecture

The cloud infrastructure has been optimized for stability and high availability:

* **Container Orchestration:** Deployed on **Amazon EKS** (Elastic Kubernetes Service).
* **Infrastructure as Code (IaC):** Configured via Terraform. The `aws_eks_node_group` has been updated to support dynamic scaling from **3 to 5 nodes** (starting with a baseline of 3 nodes) to prevent resource starvation during deployments.
* **Deployment Strategy:** Automated via **Kustomize** for dynamic Docker image tag injection from **Amazon ECR**.

---

## CI/CD Master Workflow

The deployment pipeline is fully automated and orchestrated through a single unified workflow file (`master-pipeline.yaml`) using GitHub Actions. It inherits repository secrets (`secrets: inherit`) and implements an optimized hybrid parallel-sequential architecture:

[backend-ci] ──> [backend-cd] ──> [frontend-ci] ──> [frontend-cd]

1. **`backend-ci`:** Lints, tests, and builds the backend Docker image.
2. **`backend-cd`:** Safely deletes old backend pods via `kubectl delete` to free cluster capacity, then applies new manifests using Kustomize.
3. **`frontend-ci`:** Lints, tests, and builds the frontend web client.
4. **`frontend-cd`:** Deploys the frontend client to the EKS cluster.

### Architectural Rationale & Optimization

In our architecture, the CI workflows validate code integrity before merging. We run local Docker builds to ensure no changes to the code or dependencies break container creation, without pushing to a public registry at this stage. Once approved, the CD workflows take over, compile the final image, assign official AWS ECR tags, and deploy to the Amazon EKS cluster.

To optimize performance and reduce total pipeline lead time by approximately 40%, **the Master Workflow executes the CI phases for both microservices in parallel**. However, a strict sequential dependency is maintained for the final deployment: `job-frontend-cd` requires `needs: [job-frontend-ci, job-backend-cd]`. This ensures the frontend web client is never deployed until the backend infrastructure is stable and its public endpoint is discoverable.

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


## Known Issues & Future Improvements

### 1. Terraform Destruction Deadlocks (EKS Lifecycle)
* **The Issue:** Running `terraform destroy` frequently hangs or fails when tearing down the EKS cluster. This happens because Kubernetes dynamically provisions cloud resources (such as AWS Elastic Load Balancers and Elastic Network Interfaces) that are outside of Terraform's state file. When Terraform attempts to delete the VPC or subnets, AWS blocks the operation due to these active orphan dependencies.
* **The Fix/Workaround:** Before running `terraform destroy`, all Kubernetes service resources of type `LoadBalancer` and active Ingress controllers must be manually deleted via `kubectl delete` to release the cloud network interfaces.
* **Future Migration Strategy:** To natively resolve these infrastructure dependency loops, a future improvement would be migrating the Infrastructure as Code (IaC) layer from Terraform to **AWS CDK (Cloud Development Kit) utilizing AWS EKS Blueprints**. Being a native AWS programmatic framework, CDK provides native mechanisms to better coordinate the deletion of Kubernetes-managed AWS components alongside the core infrastructure stack.




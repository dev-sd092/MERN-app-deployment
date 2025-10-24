# 🚀 MERN Application Deployment with DevSecOps on AWS

## 📘 Overview

This project demonstrates a complete DevSecOps pipeline to deploy a MERN (MongoDB, Express, React, Node.js) application on AWS EKS using a combination of Terraform, Jenkins, ArgoCD, SonarQube, Trivy, Prometheus, and Grafana.

The goal is to automate infrastructure provisioning, continuous integration, delivery, security scanning, and monitoring — all following modern industry practices.

---

## 🧱 Project Architecture

### Core Tools Used

| Category | Tools / Technologies |
|----------|---------------------|
| **Cloud Provider** | AWS |
| **Infrastructure as Code** | Terraform |
| **Containerization** | Docker |
| **Orchestration** | Amazon EKS (Elastic Kubernetes Service) |
| **CI/CD** | Jenkins, ArgoCD |
| **Version Control** | GitHub |
| **Security** | SonarQube (code quality), Trivy (image scanning) |
| **Monitoring** | Prometheus, Grafana |
| **Load Balancing** | AWS Load Balancer Controller |
| **Registry** | Amazon ECR |

---

## 🧩 Project Workflow

### 1. Infrastructure Automation using Terraform

Create S3 bucket and DynamoDB table manually (for Terraform state backend)
- DynamoDB table key: `LockID`

Initialize, validate, and apply Terraform configuration:

```bash
terraform init
terraform validate
terraform plan --var-file=terraform.tfvars
terraform apply --var-file=terraform.tfvars --auto-approve
```
<img width="1090" height="369" alt="image" src="https://github.com/user-attachments/assets/a005f8bf-10a1-4c18-b299-9861bfb16307" />
<img width="1526" height="541" alt="image" src="https://github.com/user-attachments/assets/3dfd838e-25b2-49e5-babb-c3b0aa5f9111" />
<img width="1528" height="343" alt="image" src="https://github.com/user-attachments/assets/50cce306-17b1-4e4d-9dcd-61c22366d026" />

<img width="1287" height="491" alt="image" src="https://github.com/user-attachments/assets/4f0efb57-745c-480b-bc27-b0a77167741e" />
<img width="1666" height="364" alt="image" src="https://github.com/user-attachments/assets/87b4cd1e-a0b5-4053-af29-656df287e55c" />


This provisions the required AWS resources for the application infrastructure.

---

### 2. Jenkins Server Setup

Validate installation of the following tools on the Jenkins EC2 server:
- Docker
- Jenkins
- Terraform
- Trivy
- eksctl
- kubectl
- aws-cli

Start Jenkins and install suggested plugins. Login with:
- **Username:** `admin`
- **Password:** `admin`

---
<img width="1919" height="620" alt="image" src="https://github.com/user-attachments/assets/219f3ac1-fe0c-49b5-9a7a-71c2ff56a39a" />
<img width="1918" height="1025" alt="image" src="https://github.com/user-attachments/assets/f39cf0b7-3de8-4c48-b83d-69be9e5253e0" />
<img width="1919" height="573" alt="image" src="https://github.com/user-attachments/assets/4ee481cf-2865-468e-9b57-4f1d8fa1096a" />
<img width="1919" height="897" alt="Screenshot 2025-10-24 170814" src="https://github.com/user-attachments/assets/cd8c1647-d677-41b0-a9fc-849ff18fdc65" />
<img width="1905" height="904" alt="Screenshot 2025-10-24 170835" src="https://github.com/user-attachments/assets/cd178cfe-be44-4cf2-86d9-56630b1aaf0c" />

### 3. AWS Configuration and Jenkins Integration

Configure AWS CLI on Jenkins:

```bash
aws configure
```

Install Jenkins plugins:
- AWS Credentials
- Pipeline: AWS Steps

Add credentials in Jenkins:
- AWS access key and secret key
- GitHub username and PAT
- GitHub PAT (as secret text) for updating manifests
- SonarQube token (as secret text)
- ECR repository names and AWS Account ID (as secret text)

---

### 4. EKS Cluster Setup

Create EKS Cluster:

```bash
eksctl create cluster --name Three-Tier-EKS-Cluster --region ap-south-1 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region ap-south-1 --name Three-Tier-EKS-Cluster
```
<img width="1905" height="459" alt="image" src="https://github.com/user-attachments/assets/40e81595-338e-4fe7-b4a0-3fa04d9ce30c" />
<img width="1006" height="75" alt="image" src="https://github.com/user-attachments/assets/166be43d-77a3-4277-88cd-9b9cc5e6d53f" />

---

### 5. AWS Load Balancer Controller Setup

Download IAM policy and create required resources:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster=Three-Tier-EKS-Cluster --approve
eksctl create iamserviceaccount --cluster=Three-Tier-EKS-Cluster \
--namespace=kube-system --name=aws-load-balancer-controller \
--role-name AmazonEKSLoadBalancerControllerRole \
--attach-policy-arn=arn:aws:iam::<your_account_id>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve --region=ap-south-1
```

Install Helm and deploy Load Balancer Controller:

```bash
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
--set clusterName=Three-Tier-EKS-Cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```
<img width="1778" height="342" alt="image" src="https://github.com/user-attachments/assets/ad484a41-0d0c-4c6c-b069-7eefb4e6e62a" />

---

### 6. ECR Repositories Setup

Create two private repositories in Amazon ECR:
- `frontend`
- `backend`
<img width="1668" height="433" alt="image" src="https://github.com/user-attachments/assets/9719b5b2-9569-4bd8-9097-b76903bec2bc" />
<img width="1917" height="536" alt="image" src="https://github.com/user-attachments/assets/19e6b925-0fe2-426d-84ce-60c18dc37b89" />
<img width="1909" height="519" alt="image" src="https://github.com/user-attachments/assets/32b0850d-e344-4e72-a4f0-a101e7716220" />

Configure Docker authentication with ECR:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com
```
<img width="1647" height="133" alt="image" src="https://github.com/user-attachments/assets/d7f8c313-36d0-45a2-8600-36d5c266e4b0" />

---

### 7. Kubernetes Configuration

Create namespaces:

```bash
kubectl create namespace three-tier
kubectl create namespace argocd
```

Create ECR pull secret in the `three-tier` namespace:

```bash
kubectl create secret generic ecr-registry-secret \
--from-file=.dockerconfigjson=${HOME}/.docker/config.json \
--type=kubernetes.io/dockerconfigjson --namespace three-tier
```

---

### 8. ArgoCD Installation

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Get ArgoCD credentials:

```bash
export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode)
echo $ARGO_PWD
```

Access ArgoCD UI using the LoadBalancer DNS and default username `admin`.

<img width="1919" height="1079" alt="Screenshot 2025-10-08 033253" src="https://github.com/user-attachments/assets/055f1cff-962f-425a-8363-0a60e3be28f0" />

Create ArgoCD projects for:
- frontend
- backend
- database
- ingress
<img width="1919" height="625" alt="image" src="https://github.com/user-attachments/assets/36b8be18-13d4-482f-ba82-328adf6c9200" />

Each project points to respective manifest directories.

---

### 9. SonarQube Setup

Access SonarQube on port 9000 (`http://<jenkins_ip>:9000`).

Default credentials: `admin` / `admin`

- Create tokens for both frontend and backend projects
- Configure webhooks and Jenkins integration
- Add SonarQube token in Jenkins credentials
<img width="1918" height="701" alt="image" src="https://github.com/user-attachments/assets/a3b99736-4c4f-4bf3-b5c1-6ad47cb51068" />
<img width="1896" height="1024" alt="image" src="https://github.com/user-attachments/assets/40410468-6383-48e6-9896-3fd3572ea6d3" />
<img width="1462" height="582" alt="image" src="https://github.com/user-attachments/assets/3f3a80c8-48d5-4980-ba90-79dca720a728" />

---

### 10. Jenkins Pipelines (Frontend & Backend)

Each pipeline uses the same repo, but updates different manifest files for frontend and backend.

**Pipeline stages:**
1. Checkout Code
2. SonarQube Analysis
3. Build Docker Image
4. Security Scan (Trivy)
5. Push to ECR
6. Update Manifest File
7. ArgoCD Deployment
---

**Final Output**
<img width="1919" height="1035" alt="image" src="https://github.com/user-attachments/assets/65f36d82-973c-41eb-a68c-d71608cc14d4" />
<img width="852" height="457" alt="image" src="https://github.com/user-attachments/assets/829e301c-db6f-4ede-a77a-3105bc0ccaed" />

## 🏆 Features Achieved

- ✅ Automated Infrastructure (Terraform)
- ✅ Secure CI/CD Pipelines (Jenkins + Trivy + SonarQube)
- ✅ GitOps Continuous Delivery (ArgoCD)
- ✅ AWS Managed Kubernetes (EKS)
- ✅ Centralized Monitoring (Prometheus + Grafana)
- ✅ Scalable and Modular Design (Frontend/Backend separation)

---

## 📌 Author

**Sanket Prakash Desai**  
DevOps Engineer | AWS | CI/CD | Kubernetes | Terraform

🔗 [GitHub](https://github.com/dev-sd092) | 💼 [Project Repository](https://github.com/dev-sd092/MERN-app-deployment)

---

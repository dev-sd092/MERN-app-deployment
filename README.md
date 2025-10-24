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

---

### 6. ECR Repositories Setup

Create two private repositories in Amazon ECR:
- `frontend`
- `backend`

Configure Docker authentication with ECR:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com
```

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

Create ArgoCD projects for:
- frontend
- backend
- database
- ingress

Each project points to respective manifest directories.

---

### 9. SonarQube Setup

Access SonarQube on port 9000 (`http://<jenkins_ip>:9000`).

Default credentials: `admin` / `admin`

- Create tokens for both frontend and backend projects
- Configure webhooks and Jenkins integration
- Add SonarQube token in Jenkins credentials

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

### 11. Monitoring Setup (Prometheus & Grafana)

Install Prometheus and Grafana:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install stable prometheus-community/kube-prometheus-stack
```

Expose both services:

```bash
kubectl edit svc stable-kube-prometheus-sta-prometheus
kubectl edit svc stable-grafana
```

Change `type: ClusterIP` → `type: LoadBalancer`.

Get credentials:

```bash
kubectl get secret stable-grafana -o jsonpath="{.data.admin-user}" | base64 --decode; echo
kubectl get secret stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Access Grafana UI via LoadBalancer and integrate Prometheus as a data source.

---

## 📊 Final Architecture Flow

```
Developer → GitHub → Jenkins CI/CD → ECR → ArgoCD (GitOps) → EKS Cluster
                                    ↓
                              Trivy + SonarQube
                                    ↓
                         Prometheus + Grafana (Monitoring)
```

---

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

🔗 [GitHub](https://github.com/sanketdesai) | 💼 [Project Repository](https://github.com/sanketdesai/MERN-app-deployment)

---

## 📝 License

This project is open-source and available under the [MIT License](LICENSE).

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to check the [issues page](https://github.com/sanketdesai/MERN-app-deployment/issues).

---

## ⭐ Show your support

Give a ⭐️ if this project helped you!

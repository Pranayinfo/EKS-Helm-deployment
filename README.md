# Deploying 3-tier application on AWS/GCP EKS/GKE

## Overview
This project involves the deployment of three-tier application on AWS Elastic Kubernetes Service (EKS) using Helm and Ingress for secure and scalable management. The application includes a Student App, Trainer App, and LMS Admin App, all hosted on web platforms, with iOS and Android support for the Student App. The deployment includes various AWS services such as VPC, NAT, NACL, Security Groups, Load Balancer, and AWS Certificate Manager for SSL certification. The goal is to provide a secure, robust, and efficient environment for running the application.

### High Level Architecture: Deployment
![Screenshot 2024-06-28 123123](https://github.com/Pranayinfo/EKS-Helm-deployment/assets/97338976/4761579f-7824-40ee-ad2b-4d13aac22cc2)

🚢 **High-Level Overview:**
## AWS Services Utilized

- **EKS (Elastic Kubernetes Service)**: Managed Kubernetes service for deploying containerized applications.
- **Bastion Host (EC2 instance)**: Securely manage access to your VPC.
- **ECR (Elastic Container Registry)**: Store and manage Docker container images.
- **VPC (Virtual Private Cloud)**: Isolated network for AWS resources.
- **NAT (Network Address Translation)**: Enable instances in a private subnet to connect to the internet.
- **ALB (Application Load Balancer)**: Distribute incoming application traffic across multiple targets.
- **Route53**: Scalable DNS and domain name registration.
- **AWS Certificate Manager**: Manage SSL/TLS certificates.

## Tools and Applications Installed and Configured

- **AWS CLI**: For managing AWS services.
- **Docker**: For containerizing applications.
- **Eksctl**: For creating and managing EKS clusters.
- **Kubectl**: For interacting with the Kubernetes cluster.
- **Docker Compose**: For defining and running multi-container Docker applications.
- **Ingress**: For managing external access to services in the Kubernetes cluster.
- **Helm**: For managing Kubernetes applications.

## Prerequisites
- Basic knowledge of Docker, and AWS services.
- An AWS account with necessary permissions.

## Getting Started

### Step 1: IAM Configuration
- Create a user `eks-admin` with `AdministratorAccess`.
- Generate Security Credentials: Access Key and Secret Access Key.

### Step 2: EC2 Setup
- Launch an Ubuntu instance in your favourite region (eg. region `us-west-2`).
- SSH into the instance from your local machine.

### Step 3: Install AWS CLI v2
``` shell
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure
```

### Step 4: Install Docker
``` shell
sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock
```

### Step 5: Install kubectl
``` shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

### Step 6: Install eksctl
``` shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Step 7: Setup EKS Cluster
``` shell
eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
kubectl get nodes
```

### Step 8: Install Install and configure Helm
``` shell
sudo snap install helm –classic
```
**Create the Helm Chart**
Create a Helm chart for each manifest file using the following structure:
``` shell
.
├── Chart.yaml
├── charts
│   ├── backend
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── admin-deployment.yaml
│   │   │   ├── admin-service.yaml
│   │   │   ├── auth-deployment.yaml
│   │   │   ├── auth-service.yaml
│   │   │   ├── ecr-secret.yaml
│   │   │   ├── users-deployment.yaml
│   │   │   └── users-service.yaml
│   │   └── values.yaml
│   ├── frontend
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── admin-deployment-frontend.yaml
│   │   │   ├── admin-svc-frontend.yaml
│   │   │   ├── student-deployment.yaml
│   │   │   ├── student-svc.yaml
│   │   │   ├── trainer-deployment-frontend.yaml
│   │   │   └── trainer-svc-frontend.yaml
│   │   └── values.yaml
│   ├── postgres
│   │   ├── Chart.yaml
│   │   ├── templates
│   │   │   ├── postgres-deployment.yaml
│   │   │   ├── postgres-secret.yaml
│   │   │   └── postgres-svc.yaml
│   │   └── values.yaml
│   └── redis
│       ├── Chart.yaml
│       ├── templates
│       │   ├── redis-deployment.yaml
│       │   └── redis-svc.yaml
│       └── values.yaml
├── templates
│   ├── ingress.yaml
│   ├── namespace.yaml
│   └── secrets.yaml
└── values.yaml
```

**Description of Each Helm Component and Chart**
**Chart.yaml: Metadata about the Helm chart.
values.yaml: Default values for configurable parameters.
templates Directory: Contains Kubernetes manifest templates.
	admin-deployment.yaml: Deployment for the admin backend service.
	admin-service.yaml: Service for the admin backend.
	ingress.yaml: Ingress resource for managing external access.
	namespace.yaml: Namespace for the application deployment.
	secrets.yaml: Secrets such as ECR credentials.
charts Directory: Sub-charts for each application component (backend, frontend, databases).

### Step 9: Create ECR for Images
1.	Push all the Docker images to ECR and tag them.
2.	Create the manifest file for secrets for ECR credentials.   

### Step 10: Install AWS Load Balancer
``` shell
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2
```

### Step 10: Deploy AWS Load Balancer Controller
``` shell
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller
```
### Step 11: Setting Up Ingress and Domain 
Configure path-based routing for backend APIs and frontend UI. The ingress class is ALB.
**Domain and SSL Setup**
1.	Buy Domain: From any DNS provider (e.g., GoDaddy or AWS).
2.	Route53: Create a hosted zone for the domain name and add the NS records to GoDaddy.
3.	Create Subdomain: Create a hosted zone for the subdomain.
4.	SSL Certificate: Issue a certificate from AWS Certificate Manager, which will generate the record name and value.
5.	CNAME Record: In the subdomain hosted zone, add the record name and value from the issued certificate.
6.	Add ARN in Ingress: After successfully issuing the certificate, add the ARN in the ingress.

### Step 12: Deploy the helm
``` shell
helm install version_name helm_folder_name
```

**Verify Deployment**
1.	Get Ingress ALB URL:
``` shell
kubectl get ing -n namespace
```
3.	Create CNAME Record: In the main domain hosted zone, add the subdomain name and URL.
   
**Commands for Helm and Kubernetes**
``` shell
#Helm Installation:
helm install helm-v1 excelr-helm-k8s/
#Helm Uninstall:
helm uninstall helm-v1
#Helm Rollback:
helm rollback helm-v1 1
#Helm Replication:
helm upgrade --install helm-v1 excelr-helm-k8s/
#Helm Debugging:
helm install --debug helm-v1 excelr-helm-k8s/
	
**Kubernetes Commands:**
#Get Pods:
kubectl get pods -n namespace
#Get Services:
kubectl get svc -n namespace
#Get Deployments:
kubectl get deployments -n namespace
#Describe Ingress:
kubectl describe ing ingress_name -n namespace
```

### Cleanup
- To delete the EKS cluster:
``` shell
eksctl delete cluster --name three-tier-cluster --region us-west-2
```
________________________________________
**This article outlines the comprehensive steps and configurations needed to deploy the Excelr website in a production environment using EKS, Helm, and Ingress. The detailed description ensures that all critical aspects of the deployment process are covered, providing a reliable reference for similar projects.**

---
Thank you ! 🚀👨‍💻👩‍💻

# Kubernetes Cluster Setup Guide

This guide explains different ways to create a Kubernetes cluster for learning and practice. It includes local setups such as Minikube and Kind, as well as cloud-based setups using AWS EKS through CLI, Console, and Terraform.

The examples in this repository are especially useful for learning EKS cluster creation under the Day_03/EKS Cluster Creation folder.

---

## 1. What is a Kubernetes Cluster?

A Kubernetes cluster is a group of machines that work together to run containerized applications.

It usually contains:
- one or more master/control plane nodes
- one or more worker nodes
- networking, storage, and scheduling components

A cluster lets you deploy applications, scale them, and manage failures automatically.

---

## 2. Before You Start

You will need one of the following depending on the method you choose:

### Common tools
- Docker
- kubectl
- a cloud account (for EKS, GKE, AKS)

### For local clusters
- Minikube
- Kind
- Docker Desktop (optional)

### For AWS EKS
- AWS CLI
- Terraform (if using Terraform)
- IAM access with permission to create EKS clusters

---

## 3. Option 1: Create a Local Cluster with Minikube

Minikube is one of the easiest ways to create a local Kubernetes cluster for learning.

### Install Minikube
On Linux:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

On Windows (PowerShell):

```powershell
choco install minikube -y
```

### Start the cluster

```bash
minikube start
```

### Check status

```bash
minikube status
kubectl get nodes
```

### Stop the cluster

```bash
minikube stop
```

### Delete the cluster

```bash
minikube delete
```

### Why use Minikube?
- best for beginners
- easy to install
- works on a single machine
- good for learning pods, deployments, and services

---

## 4. Option 2: Create a Local Cluster with Kind (Kubernetes in Docker)

Kind runs Kubernetes nodes inside Docker containers.

### Install Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Create a cluster

```bash
kind create cluster --name demo-cluster
```

### Verify the cluster

```bash
kubectl cluster-info
kubectl get nodes
```

### Delete the cluster

```bash
kind delete cluster --name demo-cluster
```

### Why use Kind?
- lightweight and fast
- very useful for testing and development
- great for CI/CD and local experiments

---

## 5. Option 3: Create a Cluster with kubeadm

kubeadm is a more manual way to create a Kubernetes cluster on virtual machines or servers.

### Prerequisites
- multiple Linux machines or VMs
- Docker or containerd installed
- swap disabled
- hostname resolution configured

### Basic steps
1. Install Docker/containerd
2. Install kubeadm, kubelet, and kubectl
3. Initialize the control plane
4. Join worker nodes

### Example control plane init command

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### Join worker nodes
After the init command, kubeadm gives a join command that looks like this:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

### Why use kubeadm?
- useful for understanding Kubernetes internals
- suitable for learning on VMs or bare-metal machines
- closer to a real production-like setup

---

## 6. Option 4: Create an EKS Cluster Using AWS CLI

Amazon EKS is a managed Kubernetes service from AWS.

This is one of the most common production ways to run Kubernetes.

### Install AWS CLI
Follow the official AWS CLI installation guide for your OS.

### Configure AWS credentials

```bash
aws configure
```

### Install kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### Create the EKS cluster

```bash
eksctl create cluster --name my-eks-cluster --region us-east-1 --node-type t3.medium --nodes 2
```

### Verify the cluster

```bash
kubectl get nodes
kubectl get ns
```

### Delete the cluster

```bash
eksctl delete cluster --name my-eks-cluster --region us-east-1
```

### Why use EKS CLI?
- easy to create managed Kubernetes clusters
- good for learning cloud-based Kubernetes
- widely used in real-world DevOps and cloud engineering

---

## 7. Option 5: Create an EKS Cluster Using AWS Console

You can also create an EKS cluster from the AWS Management Console.

### Steps
1. Sign in to the AWS Console.
2. Open the EKS service.
3. Click Create cluster.
4. Choose a name, Kubernetes version, and networking settings.
5. Select the node group configuration.
6. Review and create the cluster.
7. Once created, configure kubectl to connect.

### Connect to the cluster
After cluster creation, AWS will provide a command similar to:

```bash
aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster
```

### Verify

```bash
kubectl get nodes
```

### Why use the console?
- beginner-friendly
- useful for understanding the AWS EKS UI
- good for first-time learners

---

## 8. Option 6: Create an EKS Cluster Using Terraform

Terraform is a great way to create infrastructure as code.

This is useful for repeatable and professional setups.

### Install Terraform
Download and install Terraform from the official website.

### Example folder in this repo
This repository already contains Terraform examples under:
- Day_03/EKS Cluster Creation/EKS with Terraform

### Basic workflow
1. Create Terraform configuration files.
2. Initialize Terraform.
3. Apply the configuration.
4. Connect kubectl to the new cluster.

### Example commands

```bash
terraform init
terraform plan
terraform apply
```

### Connect to EKS after apply

```bash
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
```

### Destroy the infrastructure

```bash
terraform destroy
```

### Why use Terraform?
- makes infrastructure repeatable
- good for teams and production environments
- easy to version control in GitHub

---

## 9. Compare the Different Methods

| Method | Best For | Difficulty | Requires Cloud | Good For Learning |
|--------|----------|------------|---------------|-------------------|
| Minikube | Local learning | Easy | No | Yes |
| Kind | Fast local testing | Easy | No | Yes |
| kubeadm | Manual cluster setup | Medium | No | Yes |
| EKS with CLI | Managed cloud cluster | Medium | Yes | Yes |
| EKS with Console | Beginners | Easy | Yes | Yes |
| Terraform | Infrastructure as code | Medium | Yes | Yes |

---

## 10. How to Verify Your Cluster

After creating a cluster, always verify it with:

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

If these commands work, your Kubernetes cluster is ready.

---

## 11. Your First Kubernetes Deployment

Once the cluster is ready, try a simple deployment:

```bash
kubectl create deployment hello-k8s --image=nginx
kubectl expose deployment hello-k8s --type=NodePort --port=80
kubectl get pods
kubectl get svc
```

This helps beginners understand how Kubernetes runs and exposes applications.

---

## 12. Best Practice for Learners

If you are just starting, follow this learning path:
1. Start with Minikube or Kind
2. Learn pods, deployments, and services
3. Move to EKS for cloud experience
4. Practice Terraform for infrastructure automation

This path helps you understand Kubernetes step by step.

---

## 13. Summary

You can create a Kubernetes cluster in many ways:
- locally with Minikube or Kind
- manually with kubeadm
- on AWS with EKS using CLI, Console, or Terraform

For beginners, Minikube and Kind are the easiest starting points. For cloud learning, EKS is one of the best options.

---

## 14. Final Tips

- Start simple before trying production setups
- Always verify the cluster after creation
- Practice deploying a small app
- Learn kubectl commands early
- Use GitHub to store your YAML and Terraform files

With regular practice, setting up and managing Kubernetes clusters will become much easier.

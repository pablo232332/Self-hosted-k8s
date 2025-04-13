#   Self-Hosted Kubernetes Cluster on AWS (Terraform + Bash)

This project sets up a self-hosted Kubernetes cluster on AWS EC2 instances using Terraform and Bash scripts. It includes everything from provisioning infrastructure to deploying a blog application with Ingress and a Load Balancer Controller.

##   📋 Table of Contents

-   [Prerequisites](#prerequisites)
-   [Infrastructure Setup with Terraform](#infrastructure-setup-with-terraform)
-   [Accessing EC2 Instances](#accessing-ec2-instances)
-   [Setting Up Kubernetes Cluster](#setting-up-kubernetes-cluster)
-   [Installing Calico CNI](#installing-calico-cni)
-   [Configuring kubectl on the Bastion Host](#configuring-kubectl-on-the-bastion-host)
-   [Installing AWS Load Balancer Controller](#installing-aws-load-balancer-controller)
-   [Deploying the Blog App](#deploying-the-blog-app)
-   [ACM Certificate & Ingress (HTTPS)](#acm-certificate--ingress-https)
-   [Cleanup](#cleanup)

##   ✅ Prerequisites

-   An AWS account with permissions for EC2, VPC, IAM, ACM, and Route53
-   Installed locally:
    -   Terraform
    -   AWS CLI
    -   ssh, scp, tmux, kubectl
-   An AWS Key Pair (.pem file)
-   (Optional) A registered domain in Route53

##   🏗️ Infrastructure Setup with Terraform

1.  Navigate to the Terraform directory:

    ```bash
    cd tf-resources
    ```

2.  Review & update variables:

    Edit `variables.tf` or `terraform.tfvars` to change settings like region, instance type, VPC CIDR, etc.

3.  Initialize and apply:

    ```bash
    terraform init
    terraform plan
    terraform apply
    ```

    Resources created:

    -   VPC with public/private subnets, NAT, IGW
    -   EC2 Instances: 1 master, 2 workers, 1 bastion
    -   IAM Role for the load balancer controller
    -   Security Groups for the cluster and bastion host

##   🔐 Accessing EC2 Instances

1.  Copy your PEM key to the bastion:

    ```bash
    scp -i <key.pem> <key.pem> ec2-user@<BASTION_PUBLIC_IP>:~
    ```

2.  Connect to the bastion:

    ```bash
    ssh -i <key.pem> ec2-user@<BASTION_PUBLIC_IP>
    ```

3.  From bastion, SSH into nodes:

    ```bash
    ssh -i <key.pem> ec2-user@<MASTER_PRIVATE_IP>
    ssh -i <key.pem> ec2-user@<WORKER1_PRIVATE_IP>
    ssh -i <key.pem> ec2-user@<WORKER2_PRIVATE_IP>
    ```

##   ⚙️ Setting Up Kubernetes Cluster

On all nodes (master and workers):

```bash
wget [] +x create_cluster.sh
sudo ./create_cluster.sh

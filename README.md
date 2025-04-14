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
    cd resources
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


wget [] +x create_cluster.sh
sudo ./create_cluster.sh
### Prerequisites

1.  Ensure the AWS CLI and Terraform are installed and configured.
2.  Create a new key pair or have an existing one accessible.
3.  (Optional) Route 53 Domain

### Create EC2 Instances & Other Components for the Cluster Using Terraform

Resources created:

**VPC**: 2 Private subnets, 2 Public subnets, 1 nat gw, 1 igw and 2 security groups(cluster and bastion node)

**IAM role**: Permissions required to create load balancer

**EC2 instances**: 1 master, 2 worker nodes (t2.medium), and 1 Bastian host(t2.micro)

1.  cd to [tf-resources](tf-resources).
2.  Most variables are defined [here](tf-resources/1-variable.tf). Adjust them if needed.
3.  Amazon Linux image is the default AMI. Update the data resource in [here](tf-resources/4-ec2-instace.tf) for a different distro.
4.  Initialize the dir and run `terraform plan` to check for error and then `terraform apply` to create resources.
5.  Make note of IPs. Will need them to access nodes.

### Access EC2 Instances

**NOTE**: Only the bastion host is accessible from the external network. Cluster nodes reside in private subnets and can be accessed from the bastion host.

1.  Copy the PEM key to the bastion host

    `scp -i <key for ssh> <pem key> ec2-user@<bastin_public_ip>:<dst_location>`
    
3.  (Optional) Copy tmux or other personalized config to the bastion host.

4.  Connect to the bastain host via SSH.


    Install `tmux` and `git`:

    ```bash
    sudo yum update && sudo yum upgrade
    sudo hostnamectl set-hostname "bastian-node"
    sudo dnf install git tmux -y
    ```
5.  Connect to each instance via SSH from th bastion host.
6.  (Tip) Install tmux, export IPs into variables and create multiple sessions in tmux to connect to all instance at once.

    `ssh -i <>.pem ec2-user@<MASTER_IP>`

    `ssh -i <>.pem ec2-user@<WORKER1_IP>`

    `ssh -i <>.pem ec2-user@<WORKER2_IP>`

### Create Kubernetes Cluster

Use a script to create a Kubernetes cluster with kubeadm.

1.  Download the [create\_cluster](scripts/create_cluster.sh) on each node.

    ```bash
    wget [](/scripts/create_cluster.sh)
    ```
2.  Change permissions for the script.

    ```bash
    chmod +x create_cluster.sh
    ```

    NOTE: This script prepares the nodes with kubeadm as the [docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/). The cluster is intialized with pod-network-cidr=192.168.0.0/16
3.  Run the script on each node.

    ```bash
    sudo ./create_cluster.sh
    ```
4.  Select `yes` for control plane & `No` for worker nodes.

    **NOTE: Make note of cluster join command.**
5.  Install the network CNI:

    ```bash
    kubectl create -f [https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml)
    kubectl create -f [https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml)
    watch kubectl get pods -n calico-system
    ```

    NOTE: Download and update `custom-resources.yaml` with a different CIDR IF NEEDED.

6.  Join worker nodes to the cluster.

7.  Check nodes and calico status:

    ```
    kubectl get pods -n calico-system
    ```

    ```
    kubectl logs -n calico-system -l=k8s-app=calico-node
    ```

    ```
    kubect get nodes
    ```

    Exit the nodes and return to the bastion host.





### Configure kubectl on Bastian Host

1.  Add the Kubernetes repo and install `kubectl`:

    ```bash
    RELEASE="$(curl -sSL [https://dl.k8s.io/release/stable.txt](https://dl.k8s.io/release/stable.txt))"
    RELEASE="${RELEASE%.*}"
    sudo bash -c "cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=[https://pkgs.k8s.io/core:/stable:/$](https://pkgs.k8s.io/core:/stable:/$){RELEASE}/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=[https://pkgs.k8s.io/core:/stable:/$](https://pkgs.k8s.io/core:/stable:/$){RELEASE}/rpm/repodata/repomd.xml.key
    EOF"
    sudo dnf install kubectl -y
    ```
2.  Create the config dir and move the kubernetes config from the master node:

    ```bash
    mkdir -p $HOME/.kube
    scp -i .pem ec2-user@$MASTER_IP:/home/ec2-user/.kube/config $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    NOTE: change Master\_IP

### Install Load Balancer Controller

1.  Install Helm and the AWS Load Balancer Controller:

    ```
    curl -fsSL -o get_helm.sh [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)
    chmod 700 get_helm.sh
    ./get_helm.sh

    #Install aws-load-balancer controller
    helm repo add eks [https://aws.github.io/eks-charts](https://aws.github.io/eks-charts)
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=kubernetes #update cluster name if needed
    ```

    Check pods status:

    `kubectl get pods -n kube-system -l=app.kubernetes.io/instance=aws-load-balancer-controller`

    `kubectl logs -n kube-system -l=app.kubernetes.io/instance=aws-load-balancer-controller`

### Create Deployment and Services for the Blog App.

1.  Download app [manifest](manifests/app.yaml)

    `wget/manifests/app.yaml`
2.  Apply the manifest:

    `kubectl -f app.yaml`

    NOTE: Update the URLs in the configmap with your domain.
3.  Check if the app is reachable

    `curl http:<worker1 or worker 2 IP>:30011`

    `curl http:<worker1 or worker 2 IP>:30012`

### Set Up ACM certificate

Assumption: You have a domain.

1.  Go to the AWS Certificate Manager service
2.  Click **Request** and select **Request a public certificate**
3.  Provide FQDN, select **DNS validation** and select **RSA 2048**
4.  Click **Request**

### Create Ingress and load balancer

1.  Patch worker nodes with `Provider_ID`:

    `kubectl patch node <worker_node_name> -p '{"spec":{"providerID":"aws:///<Region>/<WORKER_ID>"}}'`

    * Example: `kubectl patch node worker1 -p '{"spec":{"providerID":"aws:///us-east-2/i-012373091f38897a1"}}'`
2.  Download the Ingress manifest [here](manifests/ingress.yaml)

    `wget https://raw.githubusercontent.com/gurlal-1/devops-avenue/refs/heads/main/yt-videos/k8s-aws-load-balancer/manifests/ingress.yaml`
3.  Apply the Ingress manifest:

    `kubectl create -f ingress.yaml`

    NOTE: If a domain isn't available. Remove the host and HTTPS from ingress manifest:

    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80} , {"HTTPS": 443}]'

    host: testblog.gurlal.com

    Without a domain, ACM setup won’t be applicable, and app redirects won’t work. You can access the app using the load balancer's DNS name.
4.  View Ingress logs:

    `kubectl describe ingress blog-app-ingress`

    `kubectl logs -n kube-system -l=app.kubernetes.io/instance=aws-load-balancer-controller`

    Wait for the load balancer to be created.
5.  Change the health path for the login app to `/health`
6.  Take note of the DNS name of the load balancer.

### Add a CNAME Record for the Subdomain

1.  Go to Route 53 Service. Select **Hosted Zones** and create a **new record**
2.  Enter the subdomain in the **Record name**
3.  Provide the load balancer DNS in **Value**.

### Clean up

1.  Delete Ingress

    `kubectl delete -f ingress.yaml`
2.  Exit the bastion host.
3.  Destroy the terraform resources

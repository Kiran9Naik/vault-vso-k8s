
# Create an Amazon EKS Cluster with `eksctl`

## 1. Configure AWS CLI

Make sure you have AWS credentials configured (IAM user/role with EKS permissions):


aws configure
aws sts get-caller-identity


This confirms your AWS identity.

---

## 2. Install kubectl

Download `kubectl` binary for Amazon EKS:


# Download kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.3/2025-08-03/bin/linux/amd64/kubectl

# Make it executable
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify version
kubectl version --client


---

## 3. Install eksctl

Download and install eksctl:


# Set architecture
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

# Download eksctl tarball
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" \
  | grep $PLATFORM | sha256sum --check

# Extract and install
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

# Verify installation
eksctl --version


---

## 4. Create an EKS cluster

Run the following to create a 2-node EKS cluster:


eksctl create cluster \
  --name vault-demo \
  --region ap-south-1 \
  --nodes 2 \
  --node-type c7i-flex.large \
  --managed


* `--name vault-demo` → Cluster name
* `--region ap-south-1` → AWS region
* `--nodes 2` → Number of worker nodes
* `--node-type c7i-flex.large` → EC2 instance type for worker nodes
* `--managed` → Uses managed node groups

This will take **10–15 minutes** to provision.

---

## 5. Verify the cluster

Once the cluster is up, test connectivity:


kubectl get nodes
kubectl get pods -A


---

# ⚠️ BIG NOTE — CLEANUP REQUIRED

When you’re done testing, **delete the cluster** to avoid unnecessary AWS charges:


eksctl delete cluster --name vault-demo --region ap-south-1


This deletes the EKS control plane, managed node group, and all related AWS resources created by `eksctl`.

✅ Always double-check that unused clusters, EC2 instances, EBS volumes, and load balancers are deleted in the AWS console to avoid hidden costs.

---

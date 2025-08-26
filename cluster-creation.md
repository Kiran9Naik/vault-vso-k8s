i did cluster creation of EC2 instances using eksctl 

in the existing machine i did creating those using this command:

ws configure
kubectl version --client
Client Version: v1.31.X-eks-1234567
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.3/2025-08-03/bin/linux/amd64/kubectl
kubectl version --client
Client Version: v1.31.X-eks-1234567
aws sts get-caller-identity
kubectl version --client
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
aws sts get-caller-identity

 ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
aws sts get-caller-identity
eksctl
eksctl --version

29eksctl create cluster --name vault-demo --region ap-south-1 --nodes 2 --node-type c7i-flex.large --managed
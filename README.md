## Enable ingress controller
```bash
# create an IAM OIDC provider for your cluster
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster dev-cluster --approve

# Create an IAM policy for the ALB controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# Create an IAM role and ServiceAccount for the controller
eksctl create iamserviceaccount --cluster=dev-cluster --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --region us-east-1 --approve

# Add the Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller --namespace kube-system --set clusterName=dev-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

## Enable dynamic provisioning of persistent volumes with efs

```bash
# Get VPC and subnet IDs and SG from your EKS cluster
VPC_ID=$(aws eks describe-cluster --name <your-cluster> --region <your region> --query "cluster.resourcesVpcConfig.vpcId" --output text)
SUBNET_IDS=$(aws eks describe-cluster --name <your-cluster> --region <your region> --query "cluster.resourcesVpcConfig.subnetIds" --output text | tr '\t' '\n')
EKS_NODE_SG=$(aws ec2 describe-instances --filters "Name=tag:eks:nodegroup-name,Values=<your-node-group-name>" --query "Reservations[*].Instances[*].SecurityGroups[*].GroupId" --output text --region <your region>)

# Create EFS security group
SG_ID=$(aws ec2 create-security-group --group-name "EFS-SG" --description "EFS for EKS" --vpc-id $VPC_ID --query "GroupId" --output text --region <your region>)

# Open the port 2049 on the security group to allow inbound traffic from EKS nodes
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 2049 --source-group $EKS_NODE_SG --region <your region>

# Create EFS filesystem
EFS_ID=$(aws efs create-file-system --creation-token "eks-efs" --tags "Key=Name,Value=EKS-EFS" --output text --query "FileSystemId")

# Create mount targets in each subnet
for subnet in $SUBNET_IDS; do
  aws efs create-mount-target --file-system-id $EFS_ID --subnet-id $subnet --security-groups $SG_ID --region us-east-1
done

# Associate an OIDC provider with your eks cluster
eksctl utils associate-iam-oidc-provider --cluster my-cluster --region us-east-1 --approve

# Create the IAM Role and Service Account:
eksctl create iamserviceaccount \
  --name efs-csi-controller-sa \
  --namespace kube-system \
  --cluster your-cluster-name \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --region us-east-1 \
  --approve

# Install the **EFS CSI driver** using **Helm**:
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa
```

## Rbac setup

```bash

```

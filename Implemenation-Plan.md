# Complete Sequential Guide for Setting Up EKS 1.30 with Public Subnets

## Prerequisites

# Verify AWS CLI version
aws --version

# Update kubectl for EKS 1.30
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.1/2024-01-04/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

# Get the latest eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

The error message `mv: cannot create regular file '/home/cloudshell-user/bin/': Not a directory` indicates that the destination path `/home/cloudshell-user/bin/` does not exist as a directory. This typically means that either the `bin` directory has not been created or there is a naming issue.

### Steps to Resolve

1. Create the `bin` Directory:
   First, check if the `bin` directory exists. If it doesn't, you can create it using the following command:

   
   mkdir -p $HOME/bin
   

2. Move the `eksctl` Binary:
   After creating the directory, you can move the `eksctl` binary again:

   
   sudo mv /tmp/eksctl $HOME/bin/
   

3. Update Your PATH (if necessary):
   Ensure that your `$HOME/bin` is included in your PATH environment variable so that you can run `eksctl` from anywhere. You can add it by editing your shell configuration file (like `.rc`, `._profile`, or `.zshrc`) and adding the following line:

   
   export PATH=$HOME/bin:$PATH
   

   After editing, run:

   
  source ~/.bashrc  # or the respective config file
   

4. Verify Installation:
   Finally, check if `eksctl` is correctly installed by running:

   
   eksctl version
   
# Verify installations - kubectl should be v1.30.x
kubectl version --client
eksctl version

# Create SSH key if you don't have one (optional)
ssh-keygen -t rsa -b 2048

Why? 
- kubectl version 1.30.x matches our EKS cluster version
- Latest eksctl ensures proper support for EKS 1.30
- SSH key allows secure node access for troubleshooting

# Step 1: Create IAM Roles

# 1.1 Create EKS Cluster Role

# Create cluster role trust policy
cat << EOF > cluster-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create cluster role
aws iam create-role \
  --role-name eks-cluster-role \
  --assume-role-policy-document file://cluster-trust-policy.json

# Attach required cluster policy
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
  --role-name eks-cluster-role


### 1.2 Create Node Group Role

# Create node group role trust policy
cat << EOF > node-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create node group role
aws iam create-role \
  --role-name eks-nodegroup-role \
  --assume-role-policy-document file://node-trust-policy.json

# Attach required node policies
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy \
  --role-name eks-nodegroup-role

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
  --role-name eks-nodegroup-role

aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly \
  --role-name eks-nodegroup-role


# Step 2: Create VPC with Public Subnets

# Create VPC CloudFormation template
cat << EOF > eks-vpc-public.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Amazon EKS VPC with public subnets'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: eks-vpc-public
        - Key: kubernetes.io/cluster/my-eks-cluster
          Value: shared

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: eks-igw

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: eks-public-subnet-1
        - Key: kubernetes.io/role/elb
          Value: "1"
        - Key: kubernetes.io/cluster/my-eks-cluster
          Value: shared

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: us-east-1b
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: eks-public-subnet-2
        - Key: kubernetes.io/role/elb
          Value: "1"
        - Key: kubernetes.io/cluster/my-eks-cluster
          Value: shared

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: eks-public-route-table

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable

  EksSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for EKS cluster
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: eks-cluster-sg

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"

  PublicSubnet1:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet1"

  PublicSubnet2:
    Description: Public Subnet 2 ID
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet2"
    
  SecurityGroup:
    Description: Security Group ID
    Value: !Ref EksSecurityGroup
    Export:
      Name: !Sub "${AWS::StackName}-SecurityGroup"
EOF

# Create the VPC stack
aws cloudformation create-stack \
  --stack-name eks-vpc-public \
  --template-body file://eks-vpc-public.yaml \
  --region us-east-1

# Wait for stack creation to complete
aws cloudformation wait stack-create-complete \
  --stack-name eks-vpc-public \
  --region us-east-1


# Step 3: Get Subnet IDs

# Store subnet IDs in variables
SUBNET1_ID=$(aws cloudformation describe-stacks --stack-name eks-vpc-public --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet1`].OutputValue' --output text --region us-east-1)
SUBNET2_ID=$(aws cloudformation describe-stacks --stack-name eks-vpc-public --query 'Stacks[0].Outputs[?OutputKey==`PublicSubnet2`].OutputValue' --output text --region us-east-1)

# Verify subnet IDs
echo "Subnet 1: $SUBNET1_ID"
echo "Subnet 2: $SUBNET2_ID"

Subnet 1: subnet-0deb5e992c4e3688c
Subnet 2: subnet-05eb3c8a4258a1e8d

## Step 4: Create EKS Cluster

eksctl create cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --version 1.30 \
  --vpc-public-subnets $SUBNET1_ID,$SUBNET2_ID \
  --nodegroup-name my-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size 20 \
  --managed \
  --node-private-networking=false \
  --ssh-access \
  --ssh-public-key ~/.ssh/id_rsa.pub \
  --external-dns-access \
  --asg-access \
  --full-ecr-access \
  --tags "Environment=dev,Owner=DevOps" \
  --with-oidc


## Step 5: Configure kubectl

aws eks update-kubeconfig \
  --name my-eks-cluster \
  --region us-east-1

# Verify kubectl configuration
kubectl config current-context


## Step 6: Verify Cluster

# Check cluster version
kubectl version --short

# Verify nodes are ready
kubectl get nodes

# Check running pods
kubectl get pods --all-namespaces

# Verify cluster info
kubectl cluster-info


## Step 7: Add Additional Node Labels (Optional)

# Add labels to nodes for better workload management
kubectl label nodes \
  --all \
  environment=production \
  workload-type=general


## Step 8: Deploy Sample Application (Optional)

# Create a namespace for testing
kubectl create namespace test

# Deploy sample nginx application
kubectl create deployment nginx --image=nginx -n test
kubectl expose deployment nginx --port=80 --type=LoadBalancer -n test

# Check deployment and service
kubectl get deployment nginx -n test
kubectl get service nginx -n test


## Cleanup Instructions
When you want to delete everything:

# Delete any LoadBalancer services first
kubectl delete namespace test

# Delete cluster
eksctl delete cluster --name my-eks-cluster --region us-east-1

# Delete VPC stack
aws cloudformation delete-stack --stack-name eks-vpc-public --region us-east-1

# Delete IAM roles and policies
aws iam detach-role-policy \
  --role-name eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam delete-role --role-name eks-cluster-role

aws iam detach-role-policy \
  --role-name eks-nodegroup-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam detach-role-policy \
  --role-name eks-nodegroup-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam detach-role-policy \
  --role-name eks-nodegroup-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam delete-role --role-name eks-nodegroup-role


## Important Notes:
1. All components are version-compatible with EKS 1.30
2. Using public subnets - nodes are internet-accessible
3. Enhanced security group configuration included
4. Additional cluster features enabled:
   - OIDC provider for service account IAM roles
   - Full ECR access
   - Auto Scaling Group access
   - External DNS support
5. Proper cleanup instructions to avoid stranded resources
6. All resources are in us-east-1 region

## Additional Features Added:
1. Added proper VPC and subnet tagging for EKS
2. Enhanced security group configuration
3. Optional node labeling for workload management
4. Namespace isolation for sample applications
5. OIDC provider setup for service account IAM roles
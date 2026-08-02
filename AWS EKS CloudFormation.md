AWS EKS & CloudFormation Troubleshooting - Complete Reference Guide
📋 Table of Contents
CloudFormation Stack Management

VPC Cleanup Commands

EKS Cluster Management

Complete Cleanup Scripts

Verification Commands

Error Prevention

Quick Command Reference

1. CloudFormation Stack Management
Check Stack Status
bash
# Check stack existence and status
aws cloudformation describe-stacks \
  --stack-name eksctl-spot-cluster-cluster \
  --query "Stacks[0].[StackName, StackStatus, StackStatusReason]" \
  --output table

# Check only status
aws cloudformation describe-stacks \
  --stack-name eksctl-spot-cluster-cluster \
  --query "Stacks[0].StackStatus" \
  --output text
Disable Termination Protection
bash
# Required before deletion if protection is enabled
aws cloudformation update-termination-protection \
  --stack-name eksctl-spot-cluster-cluster \
  --no-enable-termination-protection
Delete CloudFormation Stack
bash
# Standard deletion
aws cloudformation delete-stack \
  --stack-name eksctl-spot-cluster-cluster

# Delete with specific resources retained (use when resources are stuck)
aws cloudformation delete-stack \
  --stack-name eksctl-spot-cluster-cluster \
  --retain-resources VPC,InternetGateway,NATGateway

# Delete with ALL resources retained (just remove stack metadata)
aws cloudformation delete-stack \
  --stack-name eksctl-spot-cluster-cluster \
  --retain-resources "$(aws cloudformation list-stack-resources \
    --stack-name eksctl-spot-cluster-cluster \
    --query "StackResourceSummaries[].LogicalResourceId" \
    --output text | tr ' ' ',')"
Wait for Stack Deletion
bash
# Wait for completion (this can timeout with DELETE_FAILED)
aws cloudformation wait stack-delete-complete \
  --stack-name eksctl-spot-cluster-cluster

# Check if stack exists (2>&1 redirects stderr to stdout for grep)
aws cloudformation describe-stacks \
  --stack-name eksctl-spot-cluster-cluster \
  2>&1 | grep -q "does not exist" && echo "✅ Stack deleted!" || echo "❌ Stack still exists"
View Stack Events (Troubleshooting)
bash
# See recent events with failures
aws cloudformation describe-stack-events \
  --stack-name eksctl-spot-cluster-cluster \
  --max-items 10 \
  --query "StackEvents[?ResourceStatusReason].[Timestamp, LogicalResourceId, ResourceStatus, ResourceStatusReason]" \
  --output table

# See all events
aws cloudformation describe-stack-events \
  --stack-name eksctl-spot-cluster-cluster \
  --query "StackEvents[0:20].[Timestamp, LogicalResourceId, ResourceStatus]" \
  --output table
List Stack Resources
bash
# Get all resources in a stack
aws cloudformation list-stack-resources \
  --stack-name eksctl-spot-cluster-cluster \
  --query "StackResourceSummaries[].{Resource:LogicalResourceId, Type:ResourceType, Status:ResourceStatus}" \
  --output table

# Get failed resources only
aws cloudformation list-stack-resources \
  --stack-name eksctl-spot-cluster-cluster \
  --query "StackResourceSummaries[?ResourceStatus=='DELETE_FAILED'].[LogicalResourceId, PhysicalResourceId, ResourceStatusReason]" \
  --output table
2. VPC Cleanup Commands
Find VPC Dependencies
bash
VPC_ID="vpc-0de571ac753d2f0b4"

# Check ALL resources in VPC
echo "🔍 SCANNING VPC: $VPC_ID"

# 1. EC2 Instances
aws ec2 describe-instances \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Reservations[].Instances[].[InstanceId, State.Name, PrivateIpAddress, InstanceType]" \
  --output table

# 2. NAT Gateways
aws ec2 describe-nat-gateways \
  --filter "Name=vpc-id,Values=$VPC_ID" \
  --query "NatGateways[].[NatGatewayId, State, VpcId, SubnetId]" \
  --output table

# 3. Load Balancers (ALB/NLB)
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?VpcId=='$VPC_ID'].[LoadBalancerName, LoadBalancerArn, State.Code]" \
  --output table 2>/dev/null

# 4. Classic Load Balancers
aws elb describe-load-balancers \
  --query "LoadBalancerDescriptions[?VPCId=='$VPC_ID'].[LoadBalancerName]" \
  --output table 2>/dev/null

# 5. Network Interfaces (ENIs) - Most common blocker
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "NetworkInterfaces[].[NetworkInterfaceId, Status, Description, Attachment.InstanceId, PrivateIpAddress]" \
  --output table

# 6. Subnets
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[].[SubnetId, CidrBlock, AvailabilityZone, State]" \
  --output table

# 7. Internet Gateway
aws ec2 describe-internet-gateways \
  --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query "InternetGateways[].[InternetGatewayId, Attachments[].State]" \
  --output table

# 8. Route Tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "RouteTables[].[RouteTableId, Associations[0].Main]" \
  --output table

# 9. Security Groups
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "SecurityGroups[?GroupName!='default'].[GroupId, GroupName, Description]" \
  --output table

# 10. VPC Endpoints
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "VpcEndpoints[].[VpcEndpointId, VpcEndpointType, State, ServiceName]" \
  --output table

# 11. RDS Databases
aws rds describe-db-instances \
  --query "DBInstances[?DBSubnetGroup.VpcId=='$VPC_ID'].[DBInstanceIdentifier, DBInstanceStatus, Engine]" \
  --output table 2>/dev/null

# 12. ElastiCache Clusters
aws elasticache describe-cache-clusters \
  --query "CacheClusters[?CacheSubnetGroupName]" \
  --output table 2>/dev/null
Force Delete Individual Resources
bash
VPC_ID="vpc-0de571ac753d2f0b4"

# 1. Delete NAT Gateway
NAT_ID=$(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query "NatGateways[0].NatGatewayId" --output text)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
# Wait for deletion (takes ~60-90 seconds)
aws ec2 wait nat-gateway-deleted --nat-gateway-id $NAT_ID

# 2. Terminate EC2 Instances
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=vpc-id,Values=$VPC_ID" --query "Reservations[].Instances[].InstanceId" --output text)
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# 3. Delete Network Interface (ENI)
ENI_ID=$(aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID" --query "NetworkInterfaces[0].NetworkInterfaceId" --output text)
aws ec2 delete-network-interface --network-interface-id $ENI_ID
# Force delete if needed
aws ec2 delete-network-interface --network-interface-id $ENI_ID --force

# 4. Delete Load Balancer
LB_ARN=$(aws elbv2 describe-load-balancers --query "LoadBalancers[?VpcId=='$VPC_ID'].LoadBalancerArn" --output text)
aws elbv2 delete-load-balancer --load-balancer-arn $LB_ARN

# 5. Detach Internet Gateway
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 6. Delete Subnet
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
aws ec2 delete-subnet --subnet-id $SUBNET_ID

# 7. Delete Route Table (non-main)
RT_ID=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query "RouteTables[?Associations[0].Main!='true'].RouteTableId" --output text)
aws ec2 delete-route-table --route-table-id $RT_ID

# 8. Delete Security Group (non-default)
SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query "SecurityGroups[?GroupName!='default'].GroupId" --output text)
aws ec2 delete-security-group --group-id $SG_ID

# 9. Delete VPC Endpoint
ENDPOINT_ID=$(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query "VpcEndpoints[0].VpcEndpointId" --output text)
aws ec2 delete-vpc-endpoint --vpc-endpoint-id $ENDPOINT_ID

# 10. Finally, delete the VPC
aws ec2 delete-vpc --vpc-id $VPC_ID
Check VPC Status
bash
# Check if VPC exists
aws ec2 describe-vpcs --vpc-ids vpc-0de571ac753d2f0b4 2>&1

# Check all VPCs with cluster tag
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*spot-cluster*" --query "Vpcs[].VpcId" --output text

# Get VPC details
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[0].[VpcId, State, CidrBlock, IsDefault]" --output table
3. EKS Cluster Management
EKS Cluster Commands
bash
# List clusters
eksctl get cluster --region us-east-1

# List clusters in all regions
eksctl get cluster --region us-east-1 --all-regions

# Get cluster details
eksctl get cluster --name spot-cluster --region us-east-1

# Create cluster (simple)
eksctl create cluster \
  --name spot-cluster \
  --region us-east-1 \
  --nodegroup-name spot-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --spot

# Create cluster from config file
eksctl create cluster -f cluster.yaml

# Delete cluster
eksctl delete cluster --name spot-cluster --region us-east-1

# Force delete cluster (skip dependencies)
eksctl delete cluster --name spot-cluster --region us-east-1 --force

# Use unique name with timestamp
CLUSTER_NAME="spot-cluster-$(date +%Y%m%d-%H%M)"
eksctl create cluster --name $CLUSTER_NAME --region us-east-1 --spot
Kubernetes Commands
bash
# Update kubeconfig for EKS
aws eks update-kubeconfig --region us-east-1 --name spot-cluster

# Get nodes
kubectl get nodes

# Get nodes with details
kubectl get nodes -o wide

# Get pods in all namespaces
kubectl get pods --all-namespaces

# Describe cluster resources
kubectl describe nodes
4. Complete Cleanup Scripts
Script 1: Auto-Detect and Clean Stuck Stack
bash
#!/bin/bash
# save as: cleanup-stack.sh

STACK_NAME="eksctl-spot-cluster-cluster"
REGION="us-east-1"

echo "🔍 Checking stack status..."
STACK_STATUS=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $REGION \
  --query "Stacks[0].StackStatus" \
  --output text 2>&1)

if [[ "$STACK_STATUS" == *"does not exist"* ]]; then
  echo "✅ Stack does not exist. Nothing to clean."
  exit 0
fi

echo "📌 Current Status: $STACK_STATUS"

# Disable termination protection
echo "🔓 Disabling termination protection..."
aws cloudformation update-termination-protection \
  --stack-name $STACK_NAME \
  --region $REGION \
  --no-enable-termination-protection

# Try normal deletion
echo "🗑️  Attempting to delete stack..."
aws cloudformation delete-stack \
  --stack-name $STACK_NAME \
  --region $REGION

# Wait 30 seconds
sleep 30

# Check if it succeeded
NEW_STATUS=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $REGION \
  --query "Stacks[0].StackStatus" \
  --output text 2>&1)

if [[ "$NEW_STATUS" == "DELETE_IN_PROGRESS" ]]; then
  echo "✅ Stack is deleting... Waiting for completion"
  aws cloudformation wait stack-delete-complete \
    --stack-name $STACK_NAME \
    --region $REGION
  echo "✅ Stack deleted successfully!"
elif [[ "$NEW_STATUS" == "DELETE_FAILED" ]]; then
  echo "⚠️  Stack deletion failed. Finding VPC..."
  # Find VPC from stack resources
  VPC_ID=$(aws cloudformation list-stack-resources \
    --stack-name $STACK_NAME \
    --region $REGION \
    --query "StackResourceSummaries[?ResourceType=='AWS::EC2::VPC'].PhysicalResourceId" \
    --output text)
  
  if [ ! -z "$VPC_ID" ] && [ "$VPC_ID" != "None" ]; then
    echo "📌 Found VPC: $VPC_ID"
    echo "🧹 Running VPC cleanup..."
    # Call the VPC cleanup script
    ./cleanup-vpc.sh $VPC_ID
  fi
  
  # Force delete stack
  echo "🗑️  Force deleting stack..."
  aws cloudformation delete-stack \
    --stack-name $STACK_NAME \
    --region $REGION \
    --retain-resources VPC
fi
Script 2: Complete VPC Cleanup
bash
#!/bin/bash
# save as: cleanup-vpc.sh
# Usage: ./cleanup-vpc.sh <VPC_ID>

VPC_ID=$1

if [ -z "$VPC_ID" ]; then
  echo "Usage: $0 <VPC_ID>"
  exit 1
fi

echo "🔥 COMPLETE VPC CLEANUP: $VPC_ID"
echo "====================================="

# 1. Delete NAT Gateways
echo "📌 Deleting NAT Gateways..."
for nat in $(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query "NatGateways[].NatGatewayId" --output text); do
  echo "  - Deleting NAT: $nat"
  aws ec2 delete-nat-gateway --nat-gateway-id $nat
done

# Wait for NAT deletion
echo "⏳ Waiting for NAT gateways to delete (90s)..."
sleep 90

# 2. Terminate EC2 instances
echo "📌 Terminating EC2 instances..."
for instance in $(aws ec2 describe-instances --filters "Name=vpc-id,Values=$VPC_ID" --query "Reservations[].Instances[].InstanceId" --output text); do
  echo "  - Terminating: $instance"
  aws ec2 terminate-instances --instance-ids $instance
done
sleep 10

# 3. Delete Load Balancers (ALB/NLB)
echo "📌 Deleting Load Balancers..."
for lb in $(aws elbv2 describe-load-balancers --query "LoadBalancers[?VpcId=='$VPC_ID'].LoadBalancerArn" --output text 2>/dev/null); do
  echo "  - Deleting ALB/NLB: $lb"
  aws elbv2 delete-load-balancer --load-balancer-arn $lb
done

# 4. Delete Classic Load Balancers
echo "📌 Deleting Classic Load Balancers..."
for lb in $(aws elb describe-load-balancers --query "LoadBalancerDescriptions[?VPCId=='$VPC_ID'].LoadBalancerName" --output text 2>/dev/null); do
  echo "  - Deleting Classic LB: $lb"
  aws elb delete-load-balancer --load-balancer-name $lb
done
sleep 10

# 5. Delete ENIs (force)
echo "📌 Deleting ENIs..."
for eni in $(aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=$VPC_ID" --query "NetworkInterfaces[].NetworkInterfaceId" --output text); do
  echo "  - Deleting ENI: $eni"
  aws ec2 delete-network-interface --network-interface-id $eni 2>/dev/null || true
done

# 6. Delete VPC Endpoints
echo "📌 Deleting VPC Endpoints..."
for endpoint in $(aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query "VpcEndpoints[].VpcEndpointId" --output text); do
  echo "  - Deleting VPC Endpoint: $endpoint"
  aws ec2 delete-vpc-endpoint --vpc-endpoint-id $endpoint 2>/dev/null || true
done

# 7. Delete Internet Gateway
echo "📌 Detaching and deleting Internet Gateway..."
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query "InternetGateways[0].InternetGatewayId" --output text)
if [ ! -z "$IGW_ID" ] && [ "$IGW_ID" != "None" ]; then
  echo "  - Found IGW: $IGW_ID"
  aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID 2>/dev/null || true
  aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID 2>/dev/null || true
fi

# 8. Delete Subnets
echo "📌 Deleting Subnets..."
for subnet in $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[].SubnetId" --output text); do
  echo "  - Deleting subnet: $subnet"
  aws ec2 delete-subnet --subnet-id $subnet 2>/dev/null || true
done

# 9. Delete Route Tables (non-main)
echo "📌 Deleting Route Tables..."
for rt in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query "RouteTables[?Associations[0].Main!='true'].RouteTableId" --output text); do
  echo "  - Deleting route table: $rt"
  aws ec2 delete-route-table --route-table-id $rt 2>/dev/null || true
done

# 10. Delete Security Groups (non-default)
echo "📌 Deleting Security Groups..."
for sg in $(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query "SecurityGroups[?GroupName!='default'].GroupId" --output text); do
  echo "  - Deleting security group: $sg"
  aws ec2 delete-security-group --group-id $sg 2>/dev/null || true
done

# 11. Delete VPC
echo "📌 Deleting VPC: $VPC_ID"
aws ec2 delete-vpc --vpc-id $VPC_ID 2>/dev/null && echo "✅ VPC deleted!" || echo "⚠️  VPC may already be deleted"

echo "✅ Cleanup complete!"
Script 3: One-Liner Complete Cleanup
bash
#!/bin/bash
# save as: nuke-everything.sh
# USE WITH EXTREME CAUTION!

STACK_NAME="eksctl-spot-cluster-cluster"
REGION="us-east-1"

# Find VPC from stack
VPC_ID=$(aws cloudformation describe-stack-resources \
  --stack-name $STACK_NAME \
  --region $REGION \
  --query "StackResources[?ResourceType=='AWS::EC2::VPC'].PhysicalResourceId" \
  --output text 2>/dev/null)

# Delete stack
aws cloudformation update-termination-protection \
  --stack-name $STACK_NAME \
  --region $REGION \
  --no-enable-termination-protection 2>/dev/null

aws cloudformation delete-stack \
  --stack-name $STACK_NAME \
  --region $REGION \
  --retain-resources VPC 2>/dev/null

# Clean VPC if found
if [ ! -z "$VPC_ID" ] && [ "$VPC_ID" != "None" ]; then
  ./cleanup-vpc.sh $VPC_ID
fi

# Delete EKS cluster if exists
eksctl delete cluster --name spot-cluster --region $REGION --force 2>/dev/null

echo "✅ All cleanup completed!"
5. Verification Commands
Full Cleanup Verification
bash
#!/bin/bash
# save as: verify-cleanup.sh

echo "🔍 VERIFYING CLEANUP"
echo "====================="

# Check CloudFormation stack
echo -n "📌 CloudFormation Stack: "
aws cloudformation describe-stacks \
  --stack-name eksctl-spot-cluster-cluster \
  2>&1 | grep -q "does not exist" && echo "✅ Deleted" || echo "❌ Still exists"

# Check EKS cluster
echo -n "📌 EKS Cluster: "
eksctl get cluster --region us-east-1 2>&1 | grep -q "spot-cluster" && echo "❌ Still exists" || echo "✅ Deleted"

# Check VPC
VPC_ID="vpc-0de571ac753d2f0b4"
echo -n "📌 VPC ($VPC_ID): "
aws ec2 describe-vpcs --vpc-ids $VPC_ID 2>&1 | grep -q "does not exist" && echo "✅ Deleted" || echo "❌ Still exists"

# Check for any resources with cluster tag
echo "📌 Any remaining resources with 'spot-cluster' tag:"
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Name,Value=*spot-cluster* \
  --query "ResourceTagMappingList[].ResourceARN" \
  --output table 2>/dev/null | grep -v "None" || echo "  ✅ No resources found"
Generate Cleanup Report
bash
#!/bin/bash
# save as: cleanup-report.sh

echo "📊 CLEANUP STATUS REPORT"
echo "=========================="

# Stack Status
STACK_STATUS=$(aws cloudformation describe-stacks \
  --stack-name eksctl-spot-cluster-cluster \
  --query "Stacks[0].StackStatus" \
  --output text 2>&1)
echo "Stack Status: $STACK_STATUS"

# List all resources
echo -e "\nStack Resources:"
aws cloudformation list-stack-resources \
  --stack-name eksctl-spot-cluster-cluster \
  --query "StackResourceSummaries[].{Resource:LogicalResourceId, Type:ResourceType, Status:ResourceStatus}" \
  --output table 2>/dev/null

# List failed resources
echo -e "\nFailed Resources:"
aws cloudformation list-stack-resources \
  --stack-name eksctl-spot-cluster-cluster \
  --query "StackResourceSummaries[?ResourceStatus=='DELETE_FAILED'].[LogicalResourceId, PhysicalResourceId, ResourceStatusReason]" \
  --output table 2>/dev/null
6. Error Prevention
Best Practices for Cluster Creation
yaml
# Example cluster.yaml with best practices
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: spot-cluster-${CURRENT_DATE}  # Use unique name
  region: us-east-1
  version: "1.30"
  tags:
    Environment: dev
    CreatedBy: eksctl
    DeleteMe: "true"  # Helps identify resources to clean

vpc:
  cidr: 192.168.0.0/16
  nat:
    gateway: Single  # Or "Disabled" to avoid NAT Gateway costs

managedNodeGroups:
  - name: spot-nodes
    instanceType: t3.medium
    minSize: 1
    maxSize: 3
    desiredCapacity: 2
    spot: true
    tags:
      Environment: dev
Prevent Stuck CloudFormation Stacks
bash
# Always use timeout in cluster creation
eksctl create cluster -f cluster.yaml --timeout 30m

# Or use --timeout flag with CLI
eksctl create cluster \
  --name spot-cluster \
  --region us-east-1 \
  --timeout 30m \
  --spot

# Set CloudFormation timeout in config
# Add to cluster.yaml:
#   cloudFormation:
#     timeout: 30m
Quick Cleanup Aliases
bash
# Add to ~/.bashrc or ~/.zshrc

# Check stack status
alias cfn-status='aws cloudformation describe-stacks --stack-name eksctl-spot-cluster-cluster --query "Stacks[0].StackStatus" --output text'

# Delete stack with protection disabled
alias cfn-force-delete='aws cloudformation update-termination-protection --stack-name eksctl-spot-cluster-cluster --no-enable-termination-protection && aws cloudformation delete-stack --stack-name eksctl-spot-cluster-cluster'

# Show stack failures
alias cfn-failures='aws cloudformation describe-stack-events --stack-name eksctl-spot-cluster-cluster --query "StackEvents[?ResourceStatusReason].[Timestamp, LogicalResourceId, ResourceStatusReason]" --output table'

# Complete cluster cleanup
alias eks-nuke='eksctl delete cluster --name spot-cluster --region us-east-1 --force'
7. Quick Command Reference
Most Common Commands at a Glance
bash
# ===== STACK MANAGEMENT =====

# Check if stack exists
aws cloudformation describe-stacks --stack-name eksctl-spot-cluster-cluster --query "Stacks[0].StackStatus" --output text 2>&1

# Disable protection
aws cloudformation update-termination-protection --stack-name eksctl-spot-cluster-cluster --no-enable-termination-protection

# Delete stack
aws cloudformation delete-stack --stack-name eksctl-spot-cluster-cluster

# Wait for deletion
aws cloudformation wait stack-delete-complete --stack-name eksctl-spot-cluster-cluster

# Force delete with retention
aws cloudformation delete-stack --stack-name eksctl-spot-cluster-cluster --retain-resources VPC

# View stack events
aws cloudformation describe-stack-events --stack-name eksctl-spot-cluster-cluster --query "StackEvents[0:5].[Timestamp, LogicalResourceId, ResourceStatus]" --output table

# ===== VPC MANAGEMENT =====

# Check if VPC exists
aws ec2 describe-vpcs --vpc-ids vpc-0de571ac753d2f0b4 2>&1

# Find VPC ID from stack
aws cloudformation describe-stack-resources --stack-name eksctl-spot-cluster-cluster --query "StackResources[?ResourceType=='AWS::EC2::VPC'].PhysicalResourceId" --output text

# Delete VPC
aws ec2 delete-vpc --vpc-id vpc-0de571ac753d2f0b4

# Check VPC dependencies
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=vpc-0de571ac753d2f0b4" --query "NetworkInterfaces[].NetworkInterfaceId" --output text

# ===== EKS MANAGEMENT =====

# List clusters
eksctl get cluster --region us-east-1

# Delete cluster
eksctl delete cluster --name spot-cluster --region us-east-1 --force

# Create cluster (unique name)
CLUSTER_NAME="spot-cluster-$(date +%Y%m%d-%H%M)" && eksctl create cluster --name $CLUSTER_NAME --region us-east-1 --spot

# ===== VERIFICATION =====

# Full cleanup check
aws cloudformation describe-stacks --stack-name eksctl-spot-cluster-cluster 2>&1 | grep -q "does not exist" && echo "✅ Clean" || echo "❌ Still has stack"
aws ec2 describe-vpcs --vpc-ids vpc-0de571ac753d2f0b4 2>&1 | grep -q "does not exist" && echo "✅ VPC deleted" || echo "❌ VPC exists"
eksctl get cluster --region us-east-1 2>&1 | grep -q "spot-cluster" && echo "❌ Cluster exists" || echo "✅ No cluster"
📝 File Naming Convention
Save these files in a dedicated directory:

bash
mkdir ~/aws-troubleshooting
cd ~/aws-troubleshooting

# Save each script
nano cleanup-stack.sh
nano cleanup-vpc.sh
nano nuke-everything.sh
nano verify-cleanup.sh
nano cleanup-report.sh

# Make executable
chmod +x *.sh

# Create a commands reference file
cat << 'EOF' > commands-reference.txt
# Paste this entire document into commands-reference.txt
EOF
🔑 Key Takeaways
Always check stack status before deletion: aws cloudformation describe-stacks

Disable termination protection first: update-termination-protection --no-enable-termination-protection

VPC deletion fails due to dependencies (NAT Gateways, ENIs, Load Balancers)

Force delete stack with --retain-resources when resources are already deleted

Use unique cluster names to avoid collisions: spot-cluster-$(date +%Y%m%d-%H%M)

Wait for NAT Gateways to delete (takes 60-90 seconds)

Delete resources in order: NAT → Instances → ENIs → Load Balancers → IGW → Subnets → Route Tables → Security Groups → VPC

This reference guide is now complete! Save it locally and use it whenever you encounter CloudFormation/EKS stack issues.

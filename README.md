# AWS VPC CNI Installation on ROSA HCP

This guide covers installing the AWS VPC CNI on a ROSA Hosted Control Plane cluster created with `--no-cni`.

## Prerequisites

- ROSA CLI installed and logged in
- AWS CLI configured with appropriate permissions
- Helm 3.x installed
- `oc` CLI installed

## Environment Variables

Set the cluster name and VPC CIDR (used throughout this guide):
```bash
export CLUSTER_NAME=poc-cni
export VPC_CIDR=10.0.0.0/16
```

## 1. Create VPC Infrastructure (using Terraform)

First, clone the Terraform VPC example repository:
```bash
git clone https://github.com/openshift-cs/terraform-vpc-example.git terraform-vpc
```

Then create the VPC:
```bash
cd terraform-vpc
terraform init
terraform apply -var region=eu-central-1 -var cluster_name=$CLUSTER_NAME -var vpc_cidr=$VPC_CIDR -var subnet_cidr_prefix=21
export SUBNETS=$(terraform output -raw cluster-subnets-string)
# Get VPC ID from the first subnet
export VPC_ID=$(aws ec2 describe-subnets --subnet-ids $(echo $SUBNETS | cut -d',' -f1) \
  --query 'Subnets[0].VpcId' --output text)
cd ..
```

## 2. Create ROSA Cluster

```bash
# Create OIDC config
rosa create oidc-config --mode=auto
export OIDC_CONFIG_ID=<output-from-above>

# Create operator roles
rosa create operator-roles --prefix $CLUSTER_NAME --oidc-config-id $OIDC_CONFIG_ID --hosted-cp

# Create cluster with --no-cni flag
rosa create cluster --sts \
  --oidc-config-id $OIDC_CONFIG_ID \
  --operator-roles-prefix $CLUSTER_NAME \
  --hosted-cp \
  --subnet-ids=$SUBNETS \
  --no-cni \
  -c $CLUSTER_NAME \
  --machine-cidr $VPC_CIDR

# Wait for cluster to be ready (nodes will be NotReady until CNI is installed)
rosa describe cluster -c $CLUSTER_NAME

# Create admin user
rosa create admin -c $CLUSTER_NAME

# Login
oc login <api-url> -u cluster-admin -p <password>
```

## 3. Create IAM Role for VPC CNI (IRSA)

Get the OIDC provider ID:
```bash
export OIDC_PROVIDER=$(rosa describe cluster -c $CLUSTER_NAME -o json | jq -r '.aws.sts.oidc_endpoint_url' | sed 's|https://||')
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Update `network-role.json` with your OIDC provider, then create the role:
```bash
# Create the trust policy (see network-role.json)
envsubst < network-role.json > network-role-filled.json

# Create the IAM role
aws iam create-role \
  --role-name "${CLUSTER_NAME}-vpc" \
  --assume-role-policy-document file://network-role-filled.json \
  --description "IAM Role for AWS VPC CNI on ROSA"

# Attach the AWS managed EKS CNI policy
aws iam attach-role-policy \
  --role-name "${CLUSTER_NAME}-vpc" \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```

**Note**: We use the AWS managed policy `AmazonEKS_CNI_Policy` which grants all necessary EC2 permissions without restrictive tag-based conditions. Custom policies with tag conditions will NOT work because:
- The VPC CNI tags ENIs with `node.k8s.amazonaws.com/*`, not `cluster.k8s.amazonaws.com/name`
- The primary ENI (created by AWS) has no CNI-managed tags

## 4. Grant OpenShift SCCs

```bash
oc adm policy add-scc-to-user hostaccess -z aws-node -n kube-system
oc adm policy add-scc-to-user privileged -z aws-node -n kube-system
```

## 5. Update values.yaml

Edit `values.yaml` and set:
- `serviceAccount.annotations."eks.amazonaws.com/role-arn"` to your IAM role ARN
- `init.image.region` and `image.region` to your AWS region

## 6. Install AWS VPC CNI Helm Chart

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-vpc-cni \
  --namespace kube-system \
  eks/aws-vpc-cni \
  --values values.yaml
```

## 7. Patch DaemonSet for ROSA/CRI-O Paths (REQUIRED)

ROSA uses CRI-O which looks for CNI binaries and configs in different paths than containerd-based EKS:

```bash
# Patch the DaemonSet to use ROSA-compatible paths
oc patch daemonset aws-node -n kube-system --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/volumes/1/hostPath/path", "value": "/var/lib/cni/bin"},
  {"op": "replace", "path": "/spec/template/spec/volumes/2/hostPath/path", "value": "/etc/kubernetes/cni/net.d"}
]'
```

| Path Type | EKS Default | ROSA/CRI-O Path |
|-----------|-------------|-----------------|
| CNI Binaries | `/opt/cni/bin` | `/var/lib/cni/bin` |
| CNI Config | `/etc/cni/net.d` | `/etc/kubernetes/cni/net.d` |

## 8. Enable IP Forwarding (REQUIRED for each node)

IP forwarding is disabled by default on RHCOS nodes. The VPC CNI does NOT automatically enable it on ROSA. Enable it on each node:

```bash
# For each node in the cluster
for NODE in $(oc get nodes -o jsonpath='{.items[*].metadata.name}'); do
  echo "Enabling IP forwarding on $NODE..."
  oc debug node/$NODE -- chroot /host sysctl -w net.ipv4.ip_forward=1
done
```

## 9. Verify Installation

Wait for aws-node pods to be running:
```bash
oc get pods -n kube-system -l app.kubernetes.io/name=aws-node -w
```

Nodes should become Ready:
```bash
oc get nodes
```

## Validation

### Pod-to-Pod Connectivity

Create test pods and verify connectivity:
```bash
# Create test pods
oc run test-1 --image=nicolaka/netshoot -- sleep 3600
oc run test-2 --image=nicolaka/netshoot -- sleep 3600

# Wait for pods
oc get pods -w

# Check pods have VPC IPs (10.0.x.x)
oc get pods -o wide

# Test connectivity (replace <test-2-ip> with actual IP)
oc exec test-1 -- ping -c 3 <test-2-ip>
oc exec test-1 -- curl -sk https://kubernetes.default.svc/healthz
```

### External EC2 to Pod Connectivity

Verify that pods are reachable from other resources in the VPC:

```bash
# Get a subnet ID from the VPC
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0].SubnetId' --output text)

# Get the default security group (or create one that allows VPC traffic)
DEFAULT_SG=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" \
  "Name=group-name,Values=default" --query 'SecurityGroups[0].GroupId' --output text)

# Allow inbound traffic from VPC CIDR to the default security group
aws ec2 authorize-security-group-ingress --group-id $DEFAULT_SG \
  --protocol all --cidr $VPC_CIDR 2>/dev/null || true

# Create IAM role and instance profile for SSM
aws iam create-role --role-name ${CLUSTER_NAME}-ssm-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true

aws iam attach-role-policy --role-name ${CLUSTER_NAME}-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore 2>/dev/null || true

aws iam create-instance-profile --instance-profile-name ${CLUSTER_NAME}-ssm-profile 2>/dev/null || true

aws iam add-role-to-instance-profile \
  --instance-profile-name ${CLUSTER_NAME}-ssm-profile \
  --role-name ${CLUSTER_NAME}-ssm-role 2>/dev/null || true

# Wait for instance profile to propagate
sleep 10

# Create a test EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64 \
  --instance-type t3.micro \
  --subnet-id $SUBNET_ID \
  --security-group-ids $DEFAULT_SG \
  --iam-instance-profile Name=${CLUSTER_NAME}-ssm-profile \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${CLUSTER_NAME}-connectivity-test}]" \
  --query 'Instances[0].InstanceId' --output text)

echo "Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Wait for SSM agent to register (may take 1-2 minutes)
echo "Waiting for SSM agent to register..."
sleep 60

# Get a test pod IP
POD_IP=$(oc get pod test-1 -o jsonpath='{.status.podIP}')

# Test connectivity from EC2 to pod via SSM
COMMAND_ID=$(aws ssm send-command --instance-ids $INSTANCE_ID \
  --document-name "AWS-RunShellScript" \
  --parameters "commands=[\"ping -c 3 $POD_IP\"]" \
  --query 'Command.CommandId' --output text)

sleep 5
aws ssm get-command-invocation --command-id $COMMAND_ID --instance-id $INSTANCE_ID \
  --query '{Status:Status,Output:StandardOutputContent}' --output json

# Cleanup: Terminate the test instance and remove IAM resources
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws iam remove-role-from-instance-profile \
  --instance-profile-name ${CLUSTER_NAME}-ssm-profile \
  --role-name ${CLUSTER_NAME}-ssm-role 2>/dev/null || true
aws iam delete-instance-profile --instance-profile-name ${CLUSTER_NAME}-ssm-profile 2>/dev/null || true
aws iam detach-role-policy --role-name ${CLUSTER_NAME}-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore 2>/dev/null || true
aws iam delete-role --role-name ${CLUSTER_NAME}-ssm-role 2>/dev/null || true
```

## What the VPC CNI Does NOT Handle on ROSA

Unlike EKS, the VPC CNI on ROSA does **NOT** automatically configure:

| Setting | Required Action |
|---------|----------------|
| IP Forwarding | Manually enable `net.ipv4.ip_forward=1` on each node (Step 8) |

> **Note**: Source/destination check on ENIs does NOT need to be disabled - tested and verified to work with the default setting.

## What the VPC CNI Does Handle

When properly configured with IAM permissions, the VPC CNI will:
- ✅ Manage secondary ENIs for additional pod IPs
- ✅ Assign/unassign secondary private IPs
- ✅ Install CNI binaries and config files

## Troubleshooting

### Check IPAMD logs
```bash
oc debug node/<node-name> -- chroot /host tail -100 /var/log/aws-routed-eni/ipamd.log
```

### Check aws-node logs
```bash
oc logs -n kube-system -l app.kubernetes.io/name=aws-node --all-containers -f
```

### Common Issues

| Issue | Solution |
|-------|----------|
| `failed to assign an IP address` | Check IAM permissions - CNI may not be able to create secondary ENIs |
| Nodes stuck in `NotReady` | Verify DaemonSet path patches were applied |
| Pods can't communicate (100% packet loss) | Enable IP forwarding on nodes (Step 8) |
| New node pods can't communicate | Run Step 8 again for the new node |
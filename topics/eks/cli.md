# EKS — AWS CLI Commands

## Setup & Configuration

### Prerequisites

Before using the AWS CLI with EKS, ensure the following tools and permissions are in place.

**Required Tools:**
- AWS CLI v2.x (`aws --version`)
- `kubectl` (compatible with your cluster version ±1 minor version)
- `eksctl` (optional but recommended for cluster management)
- `helm` (optional, for chart deployments)

**Install/Update kubeconfig:**
```bash
aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster
```

**Verify connectivity:**
```bash
kubectl get nodes
kubectl cluster-info
```

### Required IAM Permissions

The IAM principal (user or role) needs the following managed policies or equivalent inline permissions:

| Policy | Purpose |
|---|---|
| `AmazonEKSClusterPolicy` | Manage EKS clusters |
| `AmazonEKSWorkerNodePolicy` | Node group operations |
| `AmazonEKSServicePolicy` | EKS service-linked role |
| `AmazonEC2ContainerRegistryReadOnly` | Pull images from ECR |
| `AmazonEKSVPCResourceController` | VPC CNI and security groups |

**Minimum inline policy for cluster creation:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:*",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs",
        "ec2:DescribeSecurityGroups",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PassRole",
        "cloudformation:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**Configure AWS CLI profile:**
```bash
aws configure --profile eks-admin
export AWS_PROFILE=eks-admin
export AWS_REGION=us-east-1
```

---

## Core Commands

### 1. Create an EKS Cluster

```bash
aws eks create-cluster \
  --name my-eks-cluster \
  --kubernetes-version 1.29 \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-0abc1234,subnet-0def5678,\
securityGroupIds=sg-0a1b2c3d4e5f \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --tags Environment=production,Team=platform \
  --region us-east-1
```

**What it does:** Creates a new EKS control plane. The cluster enters `CREATING` state and typically takes 10–15 minutes to become `ACTIVE`. The `--role-arn` must be an IAM role that the EKS service can assume.

**Example output:**
```json
{
    "cluster": {
        "name": "my-eks-cluster",
        "arn": "arn:aws:eks:us-east-1:123456789012:cluster/my-eks-cluster",
        "createdAt": "2024-03-15T10:30:00.000Z",
        "version": "1.29",
        "status": "CREATING",
        "roleArn": "arn:aws:iam::123456789012:role/eks-cluster-role",
        "resourcesVpcConfig": {
            "subnetIds": ["subnet-0abc1234", "subnet-0def5678"],
            "securityGroupIds": ["sg-0a1b2c3d4e5f"],
            "clusterSecurityGroupId": "sg-0z9y8x7w6v5u",
            "vpcId": "vpc-0123456789abcdef0",
            "endpointPublicAccess": true,
            "endpointPrivateAccess": false
        }
    }
}
```

---

### 2. Describe a Cluster

```bash
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1
```

**What it does:** Returns detailed metadata about a cluster including its status, Kubernetes version, VPC configuration, OIDC issuer URL, and logging configuration. Essential for verifying cluster state before performing operations.

**Example output (abbreviated):**
```json
{
    "cluster": {
        "name": "my-eks-cluster",
        "status": "ACTIVE",
        "version": "1.29",
        "endpoint": "https://ABCDEF1234567890.gr7.us-east-1.eks.amazonaws.com",
        "identity": {
            "oidc": {
                "issuer": "https://oidc.eks.us-east-1.amazonaws.com/id/ABCDEF1234567890"
            }
        },
        "certificateAuthority": {
            "data": "LS0tLS1CRUdJTi..."
        }
    }
}
```

---

### 3. List All Clusters

```bash
aws eks list-clusters \
  --region us-east-1 \
  --output table
```

**What it does:** Lists all EKS clusters in the specified region. Use `--output json` for scripting or `--output table` for human-readable output.

**Example output:**
```
---------------------------------
|         ListClusters          |
+-------------------------------+
||           clusters          ||
|+-----------------------------+|
||  my-eks-cluster             ||
||  staging-eks-cluster        ||
||  dev-eks-cluster            ||
|+-----------------------------+|
```

---

### 4. Create a Managed Node Group

```bash
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets subnet-0abc1234 subnet-0def5678 \
  --instance-types t3.medium \
  --ami-type AL2_x86_64 \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --disk-size 50 \
  --labels role=worker,environment=production \
  --tags Environment=production \
  --region us-east-1
```

**What it does:** Creates a managed node group attached to the specified cluster. AWS manages the EC2 instances, auto-scaling group, and node lifecycle. Supports both `ON_DEMAND` and `SPOT` capacity types.

---

### 5. Describe a Node Group

```bash
aws eks describe-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --region us-east-1
```

**What it does:** Returns full details of a node group including its status, scaling configuration, instance types, AMI version, health issues, and associated Auto Scaling Group.

**Example output (abbreviated):**
```json
{
    "nodegroup": {
        "nodegroupName": "my-node-group",
        "status": "ACTIVE",
        "scalingConfig": {
            "minSize": 2,
            "maxSize": 10,
            "desiredSize": 3
        },
        "instanceTypes": ["t3.medium"],
        "amiType": "AL2_x86_64",
        "releaseVersion": "1.29.3-20240322",
        "health": {
            "issues": []
        }
    }
}
```

---

### 6. List Node Groups

```bash
aws eks list-nodegroups \
  --cluster-name my-eks-cluster \
  --region us-east-1
```

**What it does:** Lists all node groups associated with a specific EKS cluster.

**Example output:**
```json
{
    "nodegroups": [
        "my-node-group",
        "spot-node-group",
        "gpu-node-group"
    ]
}
```

---

### 7. Update Cluster Version (Kubernetes Upgrade)

```bash
aws eks update-cluster-version \
  --name my-eks-cluster \
  --kubernetes-version 1.30 \
  --region us-east-1
```

**What it does:** Initiates a Kubernetes control plane version upgrade. EKS supports upgrading one minor version at a time (e.g., 1.28 → 1.29). The update ID can be used to track progress. Node groups must be upgraded separately afterward.

**Example output:**
```json
{
    "update": {
        "id": "b5f0ba18-9a87-4450-b5a0-825e6e84496f",
        "status": "InProgress",
        "type": "VersionUpdate",
        "params": [
            {
                "type": "Version",
                "value": "1.30"
            }
        ],
        "createdAt": "2024-03-15T11:00:00.000Z"
    }
}
```

---

### 8. Update a Node Group Version

```bash
aws eks update-nodegroup-version \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --kubernetes-version 1.30 \
  --force \
  --region us-east-1
```

**What it does:** Upgrades the AMI and Kubernetes version of a managed node group after the control plane has been upgraded. The `--force` flag replaces nodes even if pods cannot be gracefully evicted.

---

### 9. Update Node Group Scaling Configuration

```bash
aws eks update-nodegroup-config \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --scaling-config minSize=3,maxSize=15,desiredSize=5 \
  --region us-east-1
```

**What it does:** Modifies the scaling parameters of a managed node group without replacing nodes. Useful for scaling out ahead of anticipated load or adjusting capacity limits.

---

### 10. Create an OIDC Identity Provider

```bash
# First, get the OIDC issuer URL
OIDC_URL=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text \
  --region us-east-1)

# Associate the OIDC provider
aws eks associate-identity-provider-config \
  --cluster-name my-eks-cluster \
  --oidc provider="{issuerUrl=${OIDC_URL},clientId=sts.amazonaws.com}" \
  --region us-east-1
```

**What it does:** Associates an OIDC identity provider with the cluster, enabling IAM Roles for Service Accounts (IRSA). This allows Kubernetes pods to assume IAM roles without needing node-level permissions.

---

### 11. Create an EKS Add-on

```bash
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.1-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/eks-vpc-cni-role \
  --resolve-conflicts OVERWRITE \
  --region us-east-1
```

**What it does:** Installs a managed EKS add-on (e.g., `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver`). Managed add-ons are automatically updated and patched by AWS.

---

### 12. List Available Add-on Versions

```bash
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.29 \
  --region us-east-1 \
  --query "addons[].addonVersions[].addonVersion" \
  --output table
```

**What it does:** Lists all available versions of a specific add-on compatible with a given Kubernetes version. Use this before creating or updating add-ons to select the correct version.

---

### 13. Update kubeconfig for Cluster Access

```bash
aws eks update-kubeconfig \
  --name my-eks-cluster \
  --region us-east-1 \
  --alias my-eks-cluster-prod \
  --role-arn arn:aws:iam::123456789012:role/eks-admin-role \
  --kubeconfig ~/.kube/config
```

**What it does:** Generates or updates the local kubeconfig file to enable `kubectl` access to the cluster. The `--alias` sets a friendly context name. The `--role-arn` is used when assuming a cross-account or elevated role.

---

### 14. Delete a Node Group

```bash
aws eks delete-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-node-group \
  --region us-east-1
```

**What it does:** Initiates deletion of a managed node group. Nodes are cordoned and drained before termination. The node group must be in `ACTIVE` or `DEGRADED` state.

---

### 15. Delete a Cluster

```bash
aws eks delete-cluster \
  --name my-eks-cluster \
  --region us-east-1
```

**What it does:** Deletes an EKS cluster control plane. All node groups, Fargate profiles, and add-ons must be deleted before the cluster itself can be removed.

---

## Common Operations

### Create Operations

```bash
# Create cluster with private endpoint only
aws eks create-cluster \
  --name private-eks-cluster \
  --kubernetes-version 1.29 \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-0abc1234,subnet-0def5678,\
securityGroupIds=sg-0a1b2c3d4e5f,\
endpointPublicAccess=false,\
endpointPrivateAccess=true \
  --region us-east-1

# Create a Fargate profile
aws eks create-fargate-profile \
  --cluster-name my-eks-cluster \
  --fargate-profile-name my-fargate-profile \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/eks-fargate-role \
  --subnets subnet-0abc1234 subnet-0def5678 \
  --selectors namespace=fargate-namespace,labels={app=my-app} \
  --region us-east-1

# Create a Spot node group
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name spot-node-group \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --subnets subnet-0abc1234 subnet-0def5678 \
  --instance-types t3.medium t3.large t3a.medium \
  --capacity-type SPOT \
  --scaling-config minSize=1,maxSize=20,desiredSize=3 \
  --region us-east-1
```

---

### Read / Describe Operations

```bash
# Get cluster status only
aws eks describe-cluster \
  --name my-eks-cluster \
  --query "cluster.status" \
  --output text \
  --region us-east-1

# Get cluster endpoint
aws eks describe-cluster \
  --name my-eks-cluster \
  --query "cluster.endpoint" \
  --output text \
  --region us-east-1

# Describe a specific add-on
aws eks describe-addon \
  --cluster-name my-eks-cluster \
  --addon-name vpc-cni \
  --region us-east-1

# Describe a Fargate profile
aws eks describe-fargate-profile \
  --cluster-name my-eks-cluster \
  --fargate-profile-name my-fargate-profile \
  --region us-east-1

# Get node group health issues
aws eks describe-node
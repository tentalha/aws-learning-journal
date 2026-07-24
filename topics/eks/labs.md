# EKS — Hands-On Labs

## Lab 1: Getting Started with EKS

### Objective

In this lab, you will create your first Amazon EKS cluster using both the AWS Management Console and the AWS CLI with `eksctl`. You will deploy a sample NGINX application, expose it via a Kubernetes Service of type `LoadBalancer`, and verify end-to-end connectivity. By the end of this lab, you will understand the core components of an EKS cluster: the control plane, node groups, and how Kubernetes workloads are scheduled.

---

### Prerequisites

**AWS Services Required:**
- Amazon EKS
- Amazon EC2 (for worker nodes)
- Amazon VPC
- Elastic Load Balancing (ELB)
- IAM

**IAM Permissions Required:**
- `AmazonEKSClusterPolicy`
- `AmazonEKSWorkerNodePolicy`
- `AmazonEC2ContainerRegistryReadOnly`
- `AmazonEKS_CNI_Policy`
- `AmazonEKSServicePolicy`
- Or attach `AdministratorAccess` for lab purposes only

**Tools Required (install before starting):**
```bash
# Install eksctl
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# Install AWS CLI v2 (if not already installed)
aws --version  # Should be 2.x.x

# Configure AWS credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (e.g., us-east-1), Output format (json)
```

**Estimated Cost:** ~$0.10/hr for the EKS control plane + EC2 instance costs. Complete and clean up within 2 hours.

---

### Steps

#### Step 1: Create the EKS Cluster Using eksctl

**CLI Approach (Recommended for speed):**

```bash
# Create a cluster with a managed node group
eksctl create cluster \
  --name eks-lab-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name lab-node-group \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key ~/.ssh/id_rsa.pub \
  --asg-access \
  --full-ecr-access \
  --alb-ingress-access
```

> ⏱ **Note:** Cluster creation takes approximately **15–20 minutes**. `eksctl` will create the VPC, subnets, security groups, IAM roles, and the EKS control plane automatically.

**Console Approach:**
1. Navigate to **Amazon EKS** in the AWS Console.
2. Click **Add cluster → Create**.
3. Enter cluster name: `eks-lab-cluster`.
4. Select Kubernetes version: **1.29**.
5. For **Cluster service role**, click **Create recommended role** and follow the wizard.
6. Leave VPC settings as default or select an existing VPC.
7. Click **Next** through remaining steps and click **Create**.
8. After the cluster is `Active`, navigate to **Compute → Add node group**.
9. Node group name: `lab-node-group`, Instance type: `t3.medium`, Desired: `2`, Min: `1`, Max: `3`.

**✅ Verify Step 1:**
```bash
# Check cluster status
eksctl get cluster --name eks-lab-cluster --region us-east-1

# Update kubeconfig to connect kubectl to your cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name eks-lab-cluster

# Verify kubectl can reach the cluster
kubectl cluster-info
```

**Expected Output:**
```
Kubernetes control plane is running at https://XXXXXXXXXXXX.gr7.us-east-1.eks.amazonaws.com
CoreDNS is running at https://XXXXXXXXXXXX.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

#### Step 2: Verify Node Group Health

```bash
# Check that worker nodes are Ready
kubectl get nodes -o wide
```

**Expected Output:**
```
NAME                          STATUS   ROLES    AGE   VERSION
ip-192-168-1-10.ec2.internal  Ready    <none>   5m    v1.29.x-eks-XXXXXX
ip-192-168-2-20.ec2.internal  Ready    <none>   5m    v1.29.x-eks-XXXXXX
```

```bash
# Inspect node details
kubectl describe node ip-192-168-1-10.ec2.internal

# Check system pods are running
kubectl get pods -n kube-system
```

**Expected Output (kube-system pods):**
```
NAME                       READY   STATUS    RESTARTS   AGE
aws-node-xxxxx             1/1     Running   0          6m
coredns-xxxxxxxxx-xxxxx    1/1     Running   0          6m
coredns-xxxxxxxxx-yyyyy    1/1     Running   0          6m
kube-proxy-xxxxx           1/1     Running   0          6m
kube-proxy-yyyyy           1/1     Running   0          6m
```

---

#### Step 3: Deploy a Sample NGINX Application

```bash
# Create a namespace for our application
kubectl create namespace lab-app

# Create the deployment manifest
cat <<EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: lab-app
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
EOF

# Apply the deployment
kubectl apply -f nginx-deployment.yaml

# Watch pods come up
kubectl get pods -n lab-app -w
```

**Expected Output:**
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7d6d9b6b8d-abcde   1/1     Running   0          30s
nginx-deployment-7d6d9b6b8d-fghij   1/1     Running   0          30s
```

---

#### Step 4: Expose the Application via a LoadBalancer Service

```bash
# Create a LoadBalancer service
cat <<EOF > nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: lab-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nginx-service.yaml

# Wait for the external IP/hostname to be assigned (2-3 minutes)
kubectl get svc nginx-service -n lab-app -w
```

**Expected Output (after a few minutes):**
```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
nginx-service   LoadBalancer   10.100.45.123   a1b2c3d4e5f6g7h8i9j0.us-east-1.elb.amazonaws.com   80:32456/TCP   3m
```

```bash
# Test the application
EXTERNAL_IP=$(kubectl get svc nginx-service -n lab-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl http://$EXTERNAL_IP
```

**Expected Output:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

---

### Verification

Run the following checklist to confirm successful lab completion:

```bash
# 1. Cluster is Active
aws eks describe-cluster \
  --name eks-lab-cluster \
  --region us-east-1 \
  --query 'cluster.status'
# Expected: "ACTIVE"

# 2. Both nodes are Ready
kubectl get nodes | grep -c Ready
# Expected: 2

# 3. Pods are Running
kubectl get pods -n lab-app | grep -c Running
# Expected: 2

# 4. Service has an external endpoint
kubectl get svc nginx-service -n lab-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# Expected: Non-empty ELB hostname

# 5. Application is accessible
curl -s -o /dev/null -w "%{http_code}" \
  http://$(kubectl get svc nginx-service -n lab-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# Expected: 200
```

---

### Cleanup

> ⚠️ **Important:** Run cleanup immediately after completing the lab to avoid charges (~$0.10/hr for EKS control plane + EC2 costs).

```bash
# Step 1: Delete the Kubernetes resources first (removes the ELB)
kubectl delete namespace lab-app

# Wait 60 seconds for the ELB to be deprovisioned
sleep 60

# Step 2: Verify ELB is gone
aws elb describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancerDescriptions[*].LoadBalancerName'

# Step 3: Delete the entire cluster and all associated resources
eksctl delete cluster \
  --name eks-lab-cluster \
  --region us-east-1 \
  --wait

# Step 4: Verify cluster is deleted
eksctl get cluster --region us-east-1
# Expected: No clusters listed

# Step 5: Clean up local kubeconfig (optional)
kubectl config delete-context \
  $(kubectl config get-contexts -o name | grep eks-lab-cluster)
```

---

## Lab 2: Intermediate EKS Configuration

### Objective

In this lab, you will configure an EKS cluster with the **AWS Load Balancer Controller**, deploy a multi-tier application (frontend + backend), set up **Horizontal Pod Autoscaler (HPA)** with the Kubernetes Metrics Server, configure **Cluster Autoscaler** to automatically scale node groups, and integrate **Amazon CloudWatch Container Insights** for observability. You will also implement a **ConfigMap** and **Secret** to manage application configuration.

---

### Prerequisites

**Completed Lab 1 OR have the following ready:**
- An existing EKS cluster (version 1.29+) with OIDC enabled
- `eksctl`, `kubectl`, `helm` installed
- AWS CLI configured with appropriate permissions

**Additional Tools:**
```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Verify OIDC is enabled on your cluster
aws eks describe-cluster \
  --name eks-lab-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
# Expected: https://oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXX
```

**Additional IAM Permissions Needed:**
- `CloudWatchAgentServerPolicy`
- Custom policy for AWS Load Balancer Controller
- Custom policy for Cluster Autoscaler

---

### Steps

#### Step 1: Create a New Cluster with OIDC and IRSA Support

```bash
# Create the cluster (skip if reusing from Lab 1 — ensure --with-oidc was used)
eksctl create cluster \
  --name eks-intermediate-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name intermediate-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed \
  --with-oidc \
  --asg-access \
  --full-ecr-access \
  --alb-ingress-access

# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name eks-intermediate-cluster

# Verify
kubectl get nodes
```

---

#### Step 2: Install the AWS Load Balancer Controller

**2a. Create the IAM Policy:**

```bash
# Download the IAM policy document
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json

# Create the IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"
```

**2b. Create an IAM Service Account:**

```bash
# Create the service account with IRSA
eksctl create iamserviceaccount \
  --cluster=eks-intermediate-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region us-east-1

# Verify the service account was created
kubectl get serviceaccount aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

**2c. Install the Controller via Helm:**

```bash
# Add the EKS Helm chart repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Get the VPC ID of your cluster
VPC_ID=$(aws eks describe-cluster \
  --name eks-intermediate-cluster \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo "VPC ID: $VPC_ID"

# Install the AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-intermediate-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID

# Verify the controller is running
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**Expected Output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           60s
```

---

#### Step 3: Deploy a Multi-Tier Application

**3a. Create a Namespace and ConfigMap:**

```bash
# Create namespace
kubectl create namespace multi-tier-app

# Create a ConfigMap for application configuration
cat <<EOF > app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: multi-tier-app
data:
  APP_ENV: "staging"
  APP_PORT: "8080"
  LOG_LEVEL:
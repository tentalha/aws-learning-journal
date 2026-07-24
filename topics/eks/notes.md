# EKS

## What is it?

**Amazon Elastic Kubernetes Service (EKS)** is a fully managed Kubernetes service provided by AWS under the **Containers** category. It allows you to run Kubernetes on AWS without needing to install, operate, or maintain your own Kubernetes control plane or worker nodes.

EKS runs upstream, certified Kubernetes — meaning it is fully compatible with the standard Kubernetes ecosystem, including tools like `kubectl`, Helm, and any CNCF-conformant tooling. AWS manages the control plane (API server, etcd, scheduler, controller manager) across multiple Availability Zones, ensuring high availability and automatic patching.

EKS supports two compute modes:
- **EC2 Worker Nodes** — Self-managed or managed node groups backed by EC2 instances.
- **AWS Fargate** — Serverless compute where AWS manages the underlying infrastructure entirely.

EKS is deeply integrated with the AWS ecosystem including IAM, VPC, ALB, CloudWatch, ECR, and more.

---

## Why do we need it?

### Problems It Solves

| Problem | How EKS Solves It |
|---|---|
| Kubernetes control plane complexity | AWS manages etcd, API server, schedulers across 3 AZs |
| Cluster upgrades and patching | One-click upgrades via console or API |
| HA of the control plane | Multi-AZ control plane out of the box |
| IAM + Kubernetes RBAC integration | IRSA (IAM Roles for Service Accounts) |
| Networking complexity | VPC CNI plugin for native VPC networking |
| Cost of idle infrastructure | Fargate mode eliminates node management |

### When to Use It

- You are adopting a **microservices architecture** and need container orchestration at scale.
- Your team is already familiar with Kubernetes and wants to avoid cloud vendor lock-in.
- You need **multi-cloud portability** while still leveraging AWS services.
- You are migrating from **on-premises Kubernetes** (EKS Anywhere) to AWS.
- You need fine-grained workload isolation, resource quotas, and namespace-level governance.

### Real Business Scenarios

1. **E-commerce platform** running hundreds of microservices (cart, payment, catalog) that need independent scaling, rolling deployments, and zero-downtime updates.
2. **Machine learning pipeline** where training jobs and inference servers run as Kubernetes Jobs and Deployments, scaled by GPU node groups.
3. **SaaS company** running tenant-isolated workloads in separate namespaces with dedicated node groups per tier (free, pro, enterprise).
4. **Financial services firm** migrating from on-premises OpenShift to EKS while maintaining compliance with SOC2 and PCI-DSS.

---

## Internal Working

### Control Plane

EKS provisions a dedicated, managed Kubernetes control plane for each cluster. This control plane:

- Runs in **AWS-managed VPC** (not in your account's VPC).
- Consists of at least **2 API server instances** and **3 etcd instances** spread across **3 Availability Zones**.
- Is accessible via a **public endpoint** (default) or a **private endpoint** (via VPC endpoint).
- AWS handles certificate rotation, control plane patching, and etcd backups automatically.

```
┌──────────────────────────────────────────────────────────┐
│                   AWS-Managed VPC                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │API Server│   │API Server│   │  etcd    │             │
│  │  (AZ-1)  │   │  (AZ-2)  │   │ Cluster  │             │
│  └──────────┘   └──────────┘   └──────────┘             │
└──────────────────────────────────────────────────────────┘
           │ (ENI cross-account)
           ▼
┌──────────────────────────────────────────────────────────┐
│                 Your AWS VPC                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │  Node 1  │   │  Node 2  │   │  Node 3  │             │
│  │  (AZ-1)  │   │  (AZ-2)  │   │  (AZ-3)  │             │
│  └──────────┘   └──────────┘   └──────────┘             │
└──────────────────────────────────────────────────────────┘
```

### Data Plane

Worker nodes run in **your VPC** on EC2 instances or Fargate. They connect to the control plane via:
1. **Cross-account ENIs (Elastic Network Interfaces)** injected into your VPC.
2. The kubelet on each node communicates with the API server over these ENIs.

### Networking: VPC CNI Plugin

EKS uses the **Amazon VPC CNI plugin** (`aws-node` DaemonSet) which:
- Assigns **real VPC IP addresses** to pods (not overlay network IPs).
- Each pod gets an IP from the node's subnet.
- Uses **secondary ENIs and IP pre-warming** to reduce pod startup latency.
- Supports **prefix delegation** for high pod density.

### Authentication Flow

```
kubectl command
     │
     ▼
API Server (EKS endpoint)
     │
     ▼
AWS IAM Authenticator (webhook token review)
     │
     ├── Validates AWS STS token
     ▼
aws-auth ConfigMap / Access Entries
     │
     ▼
Kubernetes RBAC (ClusterRole / Role bindings)
```

---

## Architecture

### Core Components

```
                        ┌─────────────────────────────────────────────────┐
                        │              EKS Cluster                         │
                        │                                                   │
  kubectl / API ──────► │  ┌─────────────────────────────────────────────┐ │
                        │  │           Control Plane (AWS Managed)        │ │
                        │  │  API Server | Scheduler | Controller Manager │ │
                        │  │  etcd (3 replicas, multi-AZ)                │ │
                        │  └─────────────────────────────────────────────┘ │
                        │                      │                            │
                        │  ┌───────────────────┼───────────────────────┐  │
                        │  │          Data Plane (Your VPC)             │  │
                        │  │                                             │  │
                        │  │  ┌──────────────────────────────────────┐  │  │
                        │  │  │  Managed Node Group (EC2 ASG)         │  │  │
                        │  │  │  ┌──────────┐  ┌──────────┐          │  │  │
                        │  │  │  │  Node    │  │  Node    │          │  │  │
                        │  │  │  │ ┌──────┐ │  │ ┌──────┐ │          │  │  │
                        │  │  │  │ │ Pod  │ │  │ │ Pod  │ │          │  │  │
                        │  │  │  │ └──────┘ │  │ └──────┘ │          │  │  │
                        │  │  │  └──────────┘  └──────────┘          │  │  │
                        │  │  └──────────────────────────────────────┘  │  │
                        │  │                                             │  │
                        │  │  ┌──────────────────────────────────────┐  │  │
                        │  │  │  Fargate Pods (Serverless)            │  │  │
                        │  │  └──────────────────────────────────────┘  │  │
                        │  └─────────────────────────────────────────────┘  │
                        └─────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼──────────────────────┐
                    ▼                     ▼                       ▼
                   ECR                  ALB/NLB                  RDS
               (Container            (Ingress                (Databases)
                Registry)             Controller)
```

### Key Architectural Patterns

#### 1. Managed Node Groups
- AWS manages the lifecycle of EC2 instances (launch templates, AMIs, drain on termination).
- Backed by Auto Scaling Groups.
- Supports **spot instances** for cost savings.

#### 2. Self-Managed Node Groups
- Full control over EC2 configuration.
- You manage AMI updates, draining, and ASG configuration.
- Required for specialized hardware (e.g., GPU, Inferentia).

#### 3. Fargate Profiles
- Define which pods run on Fargate using namespace/label selectors.
- No node management, patching, or capacity planning.
- Limitations: no DaemonSets, no privileged pods, no persistent volumes (EBS).

#### 4. EKS Anywhere
- Run EKS on your own on-premises infrastructure (VMware, bare metal).
- Same EKS API and tooling, connected to AWS via EKS Connector.

#### 5. EKS Hybrid Nodes
- Connect on-premises or edge servers as worker nodes to an EKS cloud cluster.

---

## Real World Example

### Scenario: Multi-Tier E-Commerce Application on EKS

**Business Context:** An e-commerce company runs a microservices application with frontend, backend API, and worker services. They need zero-downtime deployments, auto-scaling during peak traffic, and secure access to AWS services.

#### Step 1: Create the EKS Cluster

```bash
eksctl create cluster \
  --name ecommerce-prod \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name standard-workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key my-key
```

#### Step 2: Set Up IRSA for the Backend Service

```bash
# Create IAM policy for DynamoDB access
aws iam create-policy \
  --policy-name EcommerceBackendPolicy \
  --policy-document file://backend-policy.json

# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster ecommerce-prod \
  --namespace production \
  --name backend-service-account \
  --attach-policy-arn arn:aws:iam::123456789012:policy/EcommerceBackendPolicy \
  --approve
```

#### Step 3: Deploy the Backend Service

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      serviceAccountName: backend-service-account  # IRSA
      containers:
      - name: backend-api
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/backend-api:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

#### Step 4: Configure HPA for Auto-Scaling

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

#### Step 5: Install AWS Load Balancer Controller for Ingress

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ecommerce-prod \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
  - host: api.ecommerce.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

#### Step 6: Configure Cluster Autoscaler

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=ecommerce-prod \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/ClusterAutoscalerRole
```

**Result:** The application auto-scales from 3 to 20 backend pods during peak traffic, node groups scale from 2 to 10 EC2 instances, and all pods securely access DynamoDB via IRSA without static credentials.

---

## Advantages

1. **Managed Control Plane** — Zero operational overhead for etcd, API server, and controller manager. AWS handles HA, patching, and backups automatically.

2. **Native AWS Integration** — Deep integration with IAM, VPC, ALB, EBS, EFS, ECR, CloudWatch, X-Ray, and Secrets Manager out of the box.

3. **IRSA (IAM Roles for Service Accounts)** — Fine-grained, pod-level IAM permissions without static credentials or node-level instance profiles.

4. **Upstream Kubernetes Compatibility** — Runs certified, unmodified Kubernetes. No proprietary APIs. Full ecosystem compatibility (Helm, Argo CD, Istio, etc.).

5. **Multiple Compute Options** — EC2 managed node groups, self-managed nodes, Fargate, and Karpenter for flexible cost/performance tradeoffs.

6. **Karpenter Integration** — Next-generation node provisioner that launches right-sized nodes in seconds, supporting diverse instance types and spot optimization.

7. **EKS Add-ons** — Managed lifecycle for critical components (VPC CNI, CoreDNS, kube-proxy, EBS CSI driver) with automatic version management.

8. **
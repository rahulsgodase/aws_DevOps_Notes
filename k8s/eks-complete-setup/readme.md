# 🚀 Complete EKS Setup from Scratch — Java Microservice (image:1.0)

This guide walks you **end-to-end** through setting up a complete **Amazon EKS cluster** with all essential components for deploying a **Java microservice**.

It includes:

✅ Installing CLI tools  
✅ Creating EKS cluster (with `eksctl`)  
✅ Enabling OIDC provider  
✅ Creating IAM policies (JSON)  
✅ Configuring IAM roles via IRSA  
✅ Installing core controllers:  
   - AWS Load Balancer Controller (ALB)  
   - ExternalDNS  
   - cert-manager  
   - Amazon EBS CSI driver  
✅ Deploying microservice (Deployment, Service, Ingress, HPA, StorageClass)  

---

## 🧩 Defaults & Assumptions

| Setting | Value |
|----------|-------|
| **Region** | `us-east-1` |
| **Cluster name** | `my-microservices-cluster` |
| **Namespace** | `apps` |
| **Domain** | `example.com` |
| **Hosted Zone ID** | `<ROUTE53_HOSTED_ZONE_ID>` |
| **AWS Account ID** | `<AWS_ACCOUNT_ID>` |
| **App image** | `image:1.0` |

> 🔁 Replace placeholders (`<AWS_ACCOUNT_ID>`, `<ROUTE53_HOSTED_ZONE_ID>`, `example.com`) with your own values.

---

## 🧰 A. Install Required CLIs

### Linux/macOS Quick Setup

```bash
# 1️⃣ AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 2️⃣ kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 3️⃣ eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 4️⃣ helm v3
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Verify installations:

bash
Copy code
aws --version
kubectl version --client
eksctl version
helm version
Configure AWS CLI:

bash
Copy code
aws configure
# Enter ACCESS_KEY, SECRET_KEY, region (us-east-1), output (json)
🏗️ B. Create EKS Cluster
Create eks-cluster.yaml:

yaml
Copy code
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-microservices-cluster
  region: us-east-1
  version: "1.27"
vpc:
  nat:
    gateway: HighlyAvailable
managedNodeGroups:
  - name: ng-standard
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 2
    maxSize: 6
    iam:
      withAddonPolicies:
        ebs: true
Create the cluster:

bash
Copy code
eksctl create cluster -f eks-cluster.yaml
🕒 Takes around 15–20 minutes.

🔐 C. Enable OIDC Provider
bash
Copy code
eksctl utils associate-iam-oidc-provider \
  --cluster my-microservices-cluster \
  --approve
Check OIDC:

bash
Copy code
aws eks describe-cluster --name my-microservices-cluster \
  --query "cluster.identity.oidc.issuer" --output text
🧾 D. IAM Policies
1️⃣ AWS Load Balancer Controller
bash
Copy code
curl -o iam_policy_lb_controller.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy_lb_controller.json
2️⃣ ExternalDNS Policy (externaldns-policy.json)
json
Copy code
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListHostedZonesByName",
        "route53:ListResourceRecordSets"
      ],
      "Resource": ["arn:aws:route53:::hostedzone/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["route53:ListHostedZones"],
      "Resource": ["*"]
    }
  ]
}
Create it:

bash
Copy code
aws iam create-policy \
  --policy-name ExternalDNSRoute53Policy \
  --policy-document file://externaldns-policy.json
3️⃣ EBS CSI
Use AWS managed policy:
arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

🧩 E. Create IAM Roles via IRSA
bash
Copy code
# AWS Load Balancer Controller
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve --override-existing-serviceaccounts

# ExternalDNS
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace external-dns \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/ExternalDNSRoute53Policy \
  --approve --override-existing-serviceaccounts

# EBS CSI
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve --override-existing-serviceaccounts
⚙️ F. Install Core Controllers (Helm)
1️⃣ cert-manager
bash
Copy code
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --set installCRDs=true
clusterissuer-staging.yaml:

yaml
Copy code
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - http01:
          ingress:
            class: alb
Apply:

bash
Copy code
kubectl apply -f clusterissuer-staging.yaml
2️⃣ AWS Load Balancer Controller (ALB)
bash
Copy code
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-microservices-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
3️⃣ ExternalDNS
bash
Copy code
kubectl create namespace external-dns
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install external-dns bitnami/external-dns \
  --namespace external-dns \
  --set provider=aws \
  --set aws.zoneType=public \
  --set txtOwnerId=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns
4️⃣ EBS CSI Driver
bash
Copy code
eksctl create addon \
  --cluster my-microservices-cluster \
  --name aws-ebs-csi-driver \
  --force
🧱 G. Deploy Java Microservice
Namespace
yaml
Copy code
apiVersion: v1
kind: Namespace
metadata:
  name: apps
Deployment (deployment-svc-a.yaml)
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svc-a
  namespace: apps
spec:
  replicas: 2
  selector:
    matchLabels:
      app: svc-a
  template:
    metadata:
      labels:
        app: svc-a
    spec:
      containers:
      - name: svc-a
        image: image:1.0
        ports:
        - containerPort: 8080
Service (service-svc-a.yaml)
yaml
Copy code
apiVersion: v1
kind: Service
metadata:
  name: svc-a
  namespace: apps
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: svc-a
  type: ClusterIP
Ingress (ingress-apps.yaml)
yaml
Copy code
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
  namespace: apps
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    external-dns.alpha.kubernetes.io/hostname: "svc-a.example.com"
spec:
  rules:
  - host: svc-a.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-a
            port:
              number: 80
Apply all:

bash
Copy code
kubectl apply -f apps-namespace.yaml
kubectl apply -f deployment-svc-a.yaml
kubectl apply -f service-svc-a.yaml
kubectl apply -f ingress-apps.yaml
📦 H. Optional: Storage + HPA
StorageClass (storage-ebs-sc.yaml)
yaml
Copy code
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
PVC
yaml
Copy code
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: svc-a-pvc
  namespace: apps
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi
HPA (hpa-svc-a.yaml)
yaml
Copy code
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: svc-a-hpa
  namespace: apps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: svc-a
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
🔍 I. Verify & Troubleshoot
bash
Copy code
kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -n apps
kubectl describe ingress apps-ingress -n apps
Check logs:

bash
Copy code
kubectl -n kube-system logs deploy/aws-load-balancer-controller
kubectl -n external-dns logs deploy/external-dns
🏗️ J. Build & Push Image to ECR
bash
Copy code
aws ecr create-repository --repository-name svc-a
aws ecr get-login-password | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

docker build -t svc-a:1.0 .
docker tag svc-a:1.0 <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0

kubectl set image deployment/svc-a -n apps svc-a=<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0
✅ K. Summary Checklist
Component	Status
EKS cluster	✅
OIDC provider	✅
IAM policies	✅
IRSA roles	✅
Controllers installed	✅
Java microservice deployed	✅
Ingress (ALB) working	✅
ExternalDNS + Route53	✅

🛡️ L. Production Tips
Switch Let’s Encrypt staging → production

Restrict Route53 policy to specific hosted zone ARN

Use private ECR repos & enable image scanning

Add NetworkPolicies for pod isolation

Use private subnets if service not public-facing

Regularly rotate IAM keys and audit via CloudTrail

📚 References
AWS Load Balancer Controller Docs

ExternalDNS AWS Guide

Amazon EBS CSI Driver

eksctl Official Docs

🧠 Author: Complete EKS Setup Guide for Java Microservices
💬 Feel free to fork & adapt this guide for multiple services.

yaml
Copy code

---

## ✅ How to Use This
1. Copy the above Markdown **exactly as-is**.  
2. On GitHub → `Add file → Create new file → Name it README.md`.  
3. Paste content → click **Commit changes**.  

It will render beautifully on GitHub — syntax highlighting, icons, sections, everything 🎯  

---

Would you like me to create a **folder tree + sample file names** for your GitHub repo (so you can organize manifests and policies clearly under `/infra` and `/k8s`)?  
That helps if you’ll later automate deployment via Jenkins or GitHub Actions.

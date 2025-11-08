# below is a complete, from-scratch, step-by-step EKS setup you can run right now to get a single EKS cluster with everything wired up for one Java microservice (image:1.0). 
It includes: installing CLI tools locally,

creating the EKS cluster with eksctl,

creating the OIDC provider,

IAM policies (JSON) and how to create them,

creating IAM roles and attaching to Kubernetes ServiceAccounts (IRSA) with eksctl,

installing controllers (AWS Load Balancer Controller / ALB, ExternalDNS, cert-manager, EBS CSI),

Kubernetes manifests: Namespace, Deployment (one microservice), Service, Ingress (ALB annotations), StorageClass/PVC, HPA, and cert-manager ClusterIssuer.

I’ll assume these defaults (no more clarification):

AWS Region: us-east-1

EKS cluster name: my-microservices-cluster

Kubernetes namespace for apps: apps

Domain: example.com (replace with your real domain)

Hosted zone id (Route53): <ROUTE53_HOSTED_ZONE_ID> (replace)

AWS account id: <AWS_ACCOUNT_ID> (replace)

Image: image:1.0 (as you said)

Quick notes before you run

Replace placeholders <AWS_ACCOUNT_ID> and <ROUTE53_HOSTED_ZONE_ID> and example.com with your real values.

Run these commands from a machine with admin privileges on the target AWS account.

I use eksctl, aws, kubectl, helm in commands — install them if not present (instructions below).

I’ll show both the commands and the manifest files you can apply. Ready? Let’s go.

A. Install required CLIs on your local machine

(If already installed, skip the relevant steps.)

Linux / macOS quick steps (adapt for Windows):

# 1) AWS CLI v2 (Linux example)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 2) kubectl (stable)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 3) eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 4) helm v3
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash


After installation: aws --version, kubectl version --client, eksctl version, helm version.

Configure aws cli (profile or default):

aws configure
# provide AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, default region (us-east-1), output json

B. Create EKS cluster using eksctl

Create eks-cluster.yaml (example). This will create a VPC and managed nodegroup.

eks-cluster.yaml:

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

eksctl create cluster -f eks-cluster.yaml
# note: this takes ~10-20+ minutes


This command will create the cluster and put kubeconfig in ~/.kube/config.

C. Enable OIDC provider for the cluster (needed for IRSA)
eksctl utils associate-iam-oidc-provider --cluster my-microservices-cluster --approve


Confirm OIDC provider exists:

aws eks describe-cluster --name my-microservices-cluster --query "cluster.identity.oidc.issuer" --output text

D. IAM policies — create necessary policies (LB Controller, ExternalDNS, EBS CSI)
1) AWS Load Balancer Controller policy

The AWS Load Balancer Controller maintains an official policy JSON. Download and create it as a policy:

# download official IAM policy (uses controller release path)
curl -o iam_policy_lb_controller.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy_lb_controller.json


NOTE: The above URL references a versioned copy of the policy. If you want the latest, check the controller docs. (official docs used above.)

2) ExternalDNS IAM policy (Route53)

Create a file externaldns-policy.json with this content (allows Route53 updates — tighten to specific Hosted Zone IDs if you can):

externaldns-policy.json

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
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones"
      ],
      "Resource": ["*"]
    }
  ]
}


Create the policy:

aws iam create-policy --policy-name ExternalDNSRoute53Policy --policy-document file://externaldns-policy.json


Record the returned PolicyArn for later.

3) EBS CSI: use AWS managed policy

AWS provides a managed policy AmazonEBSCSIDriverPolicy. We'll attach this managed policy to the EBS CSI service account later. No need to create custom JSON.

E. Create IAM roles & Kubernetes ServiceAccounts (IRSA) — using eksctl (recommended)

We will create 3 service accounts and bind the policy ARNs:

aws-load-balancer-controller (namespace kube-system) — attach AWSLoadBalancerControllerIAMPolicy

external-dns (namespace external-dns) — attach ExternalDNSRoute53Policy

ebs-csi-controller-sa (namespace kube-system) — attach AWS managed AmazonEBSCSIDriverPolicy

Use eksctl create iamserviceaccount (it creates the role, trust to OIDC, and annotates SA in Kubernetes).

Replace <POLICY_ARN_LB> and <POLICY_ARN_EXTDNS> with the ARNs returned from the aws iam create-policy calls, and <AWS_ACCOUNT_ID> with your account:

# 1) AWS Load Balancer Controller SA (kube-system)
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts

# 2) ExternalDNS SA (external-dns namespace)
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace external-dns \
  --name external-dns \
  --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/ExternalDNSRoute53Policy \
  --approve \
  --override-existing-serviceaccounts

# 3) EBS CSI SA (kube-system) attach AWS managed policy
eksctl create iamserviceaccount \
  --cluster my-microservices-cluster \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts


If eksctl complains that policy ARN not found, use the exact ARN returned by aws iam create-policy.

F. Install core controllers with Helm / manifests
1) cert-manager (for TLS)
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true


Create a simple ClusterIssuer for Let's Encrypt staging (testing). Save as clusterissuer-staging.yaml:

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

kubectl apply -f clusterissuer-staging.yaml

2) AWS Load Balancer Controller (ALB)

Install necessary CRDs and helm chart. Use the service account we created earlier.

# apply CRDs first (official repo)
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-microservices-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller


The service account must exist and be annotated with IAM role (eksctl handled that).

3) ExternalDNS
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


(We used serviceAccount.create=false because we created the SA via eksctl so it uses the role created).

4) Amazon EBS CSI driver

You can install EBS CSI driver as an EKS addon (if your EKS version supports it) or via Helm. Example: install as EKS add-on:

# alternative 1: eksctl to create addon
eksctl create addon --cluster my-microservices-cluster --name aws-ebs-csi-driver --force


Or via the upstream helm/manifests, ensure the ebs-csi-controller-sa is used (the eksctl created SA will let CSI call AWS APIs).

G. Kubernetes manifests — apply for 1 microservice

Create a folder k8s/ and add the following files.

1) Namespace

apps-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: apps
  labels:
    name: apps


Apply:

kubectl apply -f apps-namespace.yaml

2) Deployment (one microservice) — deployment-svc-a.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svc-a
  namespace: apps
  labels:
    app: svc-a
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
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 20
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"


Apply:

kubectl apply -f deployment-svc-a.yaml

3) Service (ClusterIP) — service-svc-a.yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-a
  namespace: apps
  labels:
    app: svc-a
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: svc-a
  type: ClusterIP


Apply:

kubectl apply -f service-svc-a.yaml

4) Ingress (ALB) — ingress-apps.yaml

Single ALB for host svc-a.example.com. ExternalDNS will create the Route53 A record using the ExternalDNS controller.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
  namespace: apps
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'
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
  tls:
  - hosts:
    - svc-a.example.com
    secretName: svc-a-tls


TLS note: above tls.secretName is a Kubernetes TLS secret. If using cert-manager + ClusterIssuer (HTTP01 via ALB), cert-manager will create the secret automatically when you annotate Ingress appropriately, or you can use certificate CRs. For a quick test you can omit TLS or use the ClusterIssuer we created earlier.

Apply:

kubectl apply -f ingress-apps.yaml

5) StorageClass + PVC (if needed) — storage-ebs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: svc-a-pvc
  namespace: apps
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi


Apply:

kubectl apply -f storage-ebs-sc.yaml

6) HPA (optional) — hpa-svc-a.yaml
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


Apply:

kubectl apply -f hpa-svc-a.yaml

H. Verify flow & troubleshooting checks

Check nodes and pods:

kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -n apps


Check Load Balancer controller logs:

kubectl -n kube-system logs deploy/aws-load-balancer-controller


Check ALB creation and Ingress:

kubectl describe ingress apps-ingress -n apps
# Once ALB created, you'll see the ALB hostname in the Ingress events/annotations.


Check ExternalDNS logs:

kubectl -n external-dns logs deploy/external-dns


Check cert-manager and Certificate status (if using):

kubectl -n cert-manager get certificates,orders,challenges
kubectl -n cert-manager logs deploy/cert-manager

I. CI/CD & image push (short)

Build and push your image (example to ECR) then update deployment to use the registry image (instead of image:1.0):

# create ECR repo and push
aws ecr create-repository --repository-name svc-a
aws ecr get-login-password | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
docker build -t svc-a:1.0 .
docker tag svc-a:1.0 <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0

# update deployment
kubectl set image deployment/svc-a -n apps svc-a=<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/svc-a:1.0


For automated pipelines use GitHub Actions / Jenkins to build → push → kubectl set image or use GitOps (ArgoCD / Flux).

J. Useful references (official docs)

AWS Load Balancer Controller install & IAM policy (official). 
kubernetes-sigs.github.io
+1

ExternalDNS AWS tutorial (for policy details). 
kubernetes-sigs.github.io

Amazon EBS CSI driver docs (permissions + install). 
AWS Documentation
+1

EKS eksctl docs (cluster creation). (You already used eksctl above.)

K. Quick checklist — what we created / wired

EKS cluster + managed nodegroup (via eksctl) ✅

OIDC provider associated for IRSA ✅

IAM policies: LB Controller (custom JSON downloaded), ExternalDNS (JSON provided), EBS CSI (AWS managed) ✅

IAM roles bound to Kubernetes ServiceAccounts via eksctl create iamserviceaccount ✅

Installed controllers: cert-manager, AWS LB Controller, ExternalDNS, EBS CSI ✅

Deployed a sample Java microservice (image:1.0) + Service + ALB Ingress + HPA + StorageClass/PVC ✅

ExternalDNS will create Route53 records (if policy and hosted zone set) ✅

L. Small security & production notes

Replace Let's Encrypt staging with production after testing. Update ClusterIssuer to production ACME server.

Don’t leave wide "arn:aws:route53:::hostedzone/*" in ExternalDNS policy if you can restrict to the specific hosted zone ARN.

Use least privilege IAM policies and rotate credentials. Use AWS Organizations / central management for Route53 in multi-account setups.

Use private ECR repos for your images and use image scanning.

Add network policies if you need pod-level isolation.

Use private subnets for worker nodes if the app must not be internet-facing; ALB can be internal if necessary.

If you want, I can do any of the following next (I’ll produce it immediately):

Paste all files in a single tar/zip (the eks-cluster.yaml, policy JSONs, all k8s/ manifests and a deploy.sh script to run the commands).

Produce a single runnable bash script (with set -e) that: creates the cluster, creates policies, creates SAs (IRSA), installs helm charts, and applies manifests — you’ll still have to paste real values for <AWS_ACCOUNT_ID> and <ROUTE53_HOSTED_ZONE_ID> and confirm you want the cluster created.

Generate the full IAM policy JSON for the ALB controller inline (it’s large) instead of downloading it from the controller repo.

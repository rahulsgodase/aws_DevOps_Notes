# 🔐 Inject AWS Secrets into Kubernetes Pods using Secrets Store CSI Driver + AWS Provider

A **complete end-to-end setup** for securely injecting secrets from **AWS Secrets Manager** into **Kubernetes pods** using the **Secrets Store CSI Driver** with the **AWS provider**.

This approach mounts secrets directly as files inside pods — **no Kubernetes Secret objects are created** (unless you enable syncing).

---

## 📘 Table of Contents

1. [Overview](#-overview)
2. [Folder Structure](#-folder-structure)
3. [Prerequisites](#-prerequisites)
4. [Step 1: Create AWS Secret](#-1️⃣-create-aws-secret)
5. [Step 2: Install Secrets Store CSI Driver](#-2️⃣-install-secrets-store-csi-driver)
6. [Step 3: Install AWS Provider for CSI Driver](#-3️⃣-install-aws-provider-for-csi-driver)
7. [Step 4: Create IAM Policy and Role for Access (IRSA)](#-4️⃣-create-iam-policy-and-role-for-access-irsa)
8. [Step 5: Configure SecretProviderClass](#-5️⃣-configure-secretproviderclass)
9. [Step 6: Create Deployment with Mounted Secrets](#-6️⃣-create-deployment-with-mounted-secrets)
10. [Step 7: (Optional) Sync Secrets as Kubernetes Secret](#-7️⃣-optional-sync-secrets-as-kubernetes-secret)
11. [Step 8: Verify Secrets in Pod](#-8️⃣-verify-secrets-in-pod)
12. [Cleanup](#-cleanup)
13. [Summary](#-summary)
14. [Maintainer](#-maintainer)

---

## 💡 Overview

This setup uses:
- **AWS Secrets Manager** → Store your database credentials (or any secret)
- **CSI Driver + AWS Provider** → Fetch secrets securely at runtime
- **IRSA (IAM Role for Service Account)** → Grant fine-grained access to Secrets Manager
- **Pod Volume Mounts** → Expose secrets as files inside pods

No static secrets in Kubernetes YAML.  
No manual sync — AWS credentials are fetched dynamically and securely.

---

## 📁 Folder Structure

```bash
aws-secrets-csi/
├── iam-policy/
│   └── csi-secrets-policy.json
├── k8s/
│   ├── secretproviderclass.yaml
│   ├── deployment.yaml
│   ├── service.yaml
└── scripts/
    └── apply.sh
```
🧩 Prerequisites
AWS CLI configured (aws configure)

kubectl, helm, and eksctl installed

EKS cluster with OIDC provider enabled:
eksctl utils associate-iam-oidc-provider --cluster <your-cluster-name> --approve

1️⃣ Create AWS Secret
```
Store DB credentials in Secrets Manager:

aws secretsmanager create-secret \
  --name mydb/credentials \
  --description "Database credentials for CSI driver test" \
  --secret-string '{"username":"admin","password":"SuperSecurePass123","host":"mydb.c9abcd123.us-east-1.rds.amazonaws.com","port":"3306"}'
  
2️⃣ Install Secrets Store CSI Driver

Install via Helm:
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update
helm install csi-driver secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system
  
Verify installation:
kubectl get pods -n kube-system | grep csi

3️⃣ Install AWS Provider for CSI Driver

Apply the AWS provider manifest:
kubectl apply -f https://github.com/aws/secrets-store-csi-driver-provider-aws/releases/latest/download/provider-aws-installer.yaml

Check:
kubectl get pods -n kube-system | grep provider-aws

4️⃣ Create IAM Policy and Role for Access (IRSA)

📄 iam-policy/csi-secrets-policy.json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:mydb/credentials*"
    }
  ]
}

Create policy:
aws iam create-policy \
  --policy-name CSISecretsAccess \
  --policy-document file://iam-policy/csi-secrets-policy.json

Create IRSA role:
eksctl create iamserviceaccount \
  --name csi-secrets-sa \
  --namespace default \
  --cluster <your-cluster-name> \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/CSISecretsAccess \
  --approve \
  --override-existing-serviceaccounts

Verify:
kubectl get sa csi-secrets-sa -o yaml

5️⃣ Configure SecretProviderClass

📄 k8s/secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-provider
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "mydb/credentials"
        objectType: "secretsmanager"
        
Apply:
kubectl apply -f k8s/secretproviderclass.yaml

6️⃣ Create Deployment with Mounted Secrets

📄 k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      serviceAccountName: csi-secrets-sa
      containers:
      - name: java-app
        image: your-dockerhub-username/java-app:1.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "aws-secrets-provider"
            
Apply:
kubectl apply -f k8s/deployment.yaml

7️⃣ (Optional) Sync Secrets as Kubernetes Secret

You can sync CSI-mounted secrets as Kubernetes Secrets automatically.

Modify your SecretProviderClass:
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "mydb/credentials"
        objectType: "secretsmanager"
  secretObjects:
  - secretName: synced-db-secret
    type: Opaque
    data:
      - objectName: "mydb/credentials"
        key: db-creds

Apply:
kubectl apply -f k8s/secretproviderclass.yaml

Check:
kubectl get secret synced-db-secret -o yaml

8️⃣ Verify Secrets in Pod

Check that the secrets are mounted properly:
kubectl exec -it deploy/java-app -- ls /mnt/secrets-store

Expected output:
mydb/credentials

View the secret file contents:
kubectl exec -it deploy/java-app -- cat /mnt/secrets-store/mydb/credentials

🧹 Cleanup
kubectl delete -f k8s/
aws secretsmanager delete-secret --secret-id mydb/credentials --force-delete-without-recovery
aws iam delete-policy --policy-arn arn:aws:iam::<account-id>:policy/CSISecretsAccess

📊 Summary
Step	Description
1️⃣	Create secret in AWS Secrets Manager
2️⃣	Install Secrets Store CSI Driver
3️⃣	Install AWS Provider
4️⃣	Create IAM policy + IRSA role
5️⃣	Define SecretProviderClass
6️⃣	Deploy app and mount secret
7️⃣	(Optional) Sync as K8s Secret
8️⃣	Verify inside pod
9️⃣	Cleanup


🚀 Quick Apply Script

📄 scripts/apply.sh
#!/bin/bash
kubectl apply -f k8s/secretproviderclass.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

Make it executable:
chmod +x scripts/apply.sh

Run:
./scripts/apply.sh
```










# 🧩 Inject Database Credentials into Kubernetes Pod using AWS Secrets Manager & External Secrets Operator (ESO)

A complete step-by-step setup to securely inject **database credentials (username, password, endpoint, port)** from **AWS Secrets Manager** into a **Kubernetes Pod** using **External Secrets Operator (ESO)** and **IAM Role for Service Accounts (IRSA)**.


## 📘 Table of Contents
1. [Overview](#-overview)
2. [Folder Structure](#-folder-structure)
3. [Prerequisites](#-prerequisites)
4. [Step 1: Create Secret in AWS Secrets Manager](#-1️⃣-create-secret-in-aws-secrets-manager)
5. [Step 2: Install External Secrets Operator](#-2️⃣-install-external-secrets-operator)
6. [Step 3: Create IAM Policy for Access](#-3️⃣-create-iam-policy-for-access)
7. [Step 4: Create IAM Role for ServiceAccount (IRSA)](#-4️⃣-create-iam-role-for-serviceaccount-irsa)
8. [Step 5: Configure SecretStore](#-5️⃣-configure-secretstore)
9. [Step 6: Create ExternalSecret](#-6️⃣-create-externalsecret)
10. [Step 7: Create Deployment with Secrets as Env Vars](#-7️⃣-create-deployment-with-secrets-as-env-vars)
11. [Step 8: Create Service (Optional)](#-8️⃣-create-service-optional)
12. [Step 9: Verify Everything](#-9️⃣-verify-everything)
13. [Step 10: Rotate Secrets Automatically](#-10️⃣-rotate-secrets-automatically)
14. [Cleanup](#-cleanup)
15. [Summary](#-summary)
16. [Maintainer](#-maintainer)


## 💡 Overview

This setup ensures your pods get database credentials **securely** and **dynamically** from AWS Secrets Manager.  
It removes the need to hardcode secrets in Kubernetes YAML.

**Key components:**
- AWS Secrets Manager (stores DB credentials)
- External Secrets Operator (syncs AWS → K8s)
- IAM Role for Service Account (IRSA)
- Kubernetes Deployment consuming secrets as environment variables


# 📁 Folder Structure
```
db-secret-setup/
├── iam-policy/
│   └── external-secrets-policy.json
├── k8s/
│   ├── secretproviderclass.yaml
│   ├── externalsecret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
└── scripts/
    └── apply.sh
```
🧩 Prerequisites
AWS CLI configured (aws configure)

kubectl, helm, and eksctl installed

Existing EKS cluster with OIDC provider enabled
eksctl utils associate-iam-oidc-provider --cluster <your-cluster-name> --approve

# 1️⃣ Create Secret in AWS Secrets Manager

``` 
aws secretsmanager create-secret \
  --name mydb/credentials \
  --description "Database credentials for app" \
  --secret-string '{"username":"admin","password":"P@ssw0rd123","host":"mydb.c9abcd123.us-east-1.rds.amazonaws.com","port":"3306"}'
```
# 2️⃣ Install External Secrets Operator
```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
  
Check installation:
kubectl get pods -n external-secrets
```
# 3️⃣ Create IAM Policy for Access
```
📄 iam-policy/external-secrets-policy.json

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

Create the policy in AWS:
aws iam create-policy \
  --policy-name ExternalSecretsAccess \
  --policy-document file://iam-policy/external-secrets-policy.json
 ``` 
# 4️⃣ Create IAM Role for ServiceAccount (IRSA)
```
eksctl create iamserviceaccount \
  --name external-secrets-sa \
  --namespace external-secrets \
  --cluster <your-cluster-name> \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/ExternalSecretsAccess \
  --approve \
  --override-existing-serviceaccounts
  
Check:
kubectl get sa external-secrets-sa -n external-secrets -o yaml
```
# 5️⃣ Configure SecretStore
```
📄 k8s/secretproviderclass.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            
Apply:
kubectl apply -f k8s/secretproviderclass.yaml
```
# 6️⃣ Create ExternalSecret
```
📄 k8s/externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: mydb/credentials
        property: username
    - secretKey: password
      remoteRef:
        key: mydb/credentials
        property: password
    - secretKey: host
      remoteRef:
        key: mydb/credentials
        property: host
    - secretKey: port
      remoteRef:
        key: mydb/credentials
        property: port
        
Apply:
kubectl apply -f k8s/externalsecret.yaml

Verify secret synced:
kubectl get secret db-secret -o yaml
```
# 7️⃣ Create Deployment with Secrets as Env Vars
```
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
      serviceAccountName: external-secrets-sa
      containers:
      - name: java-app
        image: your-dockerhub-username/java-app:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: port
              
Apply:
kubectl apply -f k8s/deployment.yaml
```
# 8️⃣ Create Service (Optional)
```
📄 k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: java-app-svc
spec:
  selector:
    app: java-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
  
Apply:
kubectl apply -f k8s/service.yaml
```
# 9️⃣ Verify Everything
```
Check ESO status:
kubectl get externalsecret
kubectl describe externalsecret db-credentials

Check synced secret:
kubectl get secret db-secret -o yaml

Check environment variables in pod:
kubectl exec -it deploy/java-app -- env | grep DB_

🔄 10️⃣ Rotate Secrets Automatically

ESO auto-syncs when you update the AWS secret.
aws secretsmanager update-secret \
  --secret-id mydb/credentials \
  --secret-string '{"username":"admin","password":"NewSecureP@ssword","host":"mydb.c9abcd123.us-east-1.rds.amazonaws.com","port":"3306"}'
Within a few minutes, Kubernetes secrets update automatically.

🧹 Cleanup

kubectl delete -f k8s/
aws secretsmanager delete-secret --secret-id mydb/credentials --force-delete-without-recovery
aws iam delete-policy --policy-arn arn:aws:iam::<account-id>:policy/ExternalSecretsAccess

📊 Summary
Step	Description

1️⃣	Create secret in AWS Secrets Manager
2️⃣	Install External Secrets Operator
3️⃣	Create IAM policy for access
4️⃣	Create IRSA-enabled service account
5️⃣	Configure SecretStore
6️⃣	Create ExternalSecret to sync
7️⃣	Inject into pods via env vars
8️⃣	Verify + Rotate secrets
9️⃣	Cleanup when done
```

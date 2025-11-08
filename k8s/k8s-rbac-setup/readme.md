# 🧩 Kubernetes RBAC with AWS IAM User (Complete Setup from Scratch)

This repository demonstrates how to **grant granular Kubernetes access** to an **AWS IAM user** using **RBAC (Role-Based Access Control)** and the **`aws-auth` ConfigMap** in an Amazon EKS cluster.

---

## 🎯 Objective

To configure an AWS IAM user with restricted Kubernetes permissions using:

- IAM user mapping in `aws-auth`
- Kubernetes Roles / ClusterRoles
- RoleBindings / ClusterRoleBindings
- Verification via `kubectl auth can-i`

Works seamlessly on **EKS**, and conceptually similar on **AKS / GKE**.

---

## 📁 Folder Structure
```
rbac/
├── namespace.yaml
├── role.yaml
├── rolebinding-user.yaml
├── rolebinding-group.yaml
├── clusterrole.yaml
├── clusterrolebinding.yaml
scripts/
└── apply-rbac.sh
```

yaml
Copy code

---

## 🚀 Step-by-Step Setup

### 1️⃣ Create IAM User in AWS

```bash
aws iam create-user --user-name dev-rahul
Attach minimal permissions (required to describe EKS clusters):

bash
Copy code
aws iam attach-user-policy \
  --user-name dev-rahul \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSReadOnlyAccess

Or create a minimal custom policy:

json
Copy code
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["eks:DescribeCluster"], "Resource": "*" },
    { "Effect": "Allow", "Action": ["sts:GetCallerIdentity"], "Resource": "*" }
  ]
}

### 2️ Configure IAM User Locally

Generate access keys and configure a local AWS CLI profile:

bash
Copy code
aws iam create-access-key --user-name dev-rahul
aws configure --profile dev-rahul
# Enter Access Key, Secret, Region, Output Format

### 3️⃣ Map IAM User in EKS aws-auth ConfigMap
Edit the aws-auth ConfigMap in your EKS cluster:

bash
Copy code
kubectl -n kube-system edit configmap aws-auth
Add this entry under mapUsers:

yaml
Copy code
mapUsers: |
  - userarn: arn:aws:iam::<AWS_ACCOUNT_ID>:user/dev-rahul
    username: dev-rahul
    groups:
      - devs
✅ This maps your AWS IAM user dev-rahul to the Kubernetes user dev-rahul and assigns them to the devs group.

### 4️⃣ Apply Kubernetes RBAC YAMLs
rbac/namespace.yaml
yaml
Copy code
apiVersion: v1
kind: Namespace
metadata:
  name: rbac-demo


rbac/role.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-demo
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]


rbac/rolebinding-group.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding-group
  namespace: rbac-demo
subjects:
- kind: Group
  name: devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io


rbac/clusterrole.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services", "namespaces"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]


rbac/clusterrolebinding.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-read-only-binding
subjects:
- kind: Group
  name: devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-read-only
  apiGroup: rbac.authorization.k8s.io

⚙️ 5️⃣ Apply Everything with One Script
scripts/apply-rbac.sh
bash
Copy code
#!/bin/bash
set -euo pipefail

echo "Applying Kubernetes RBAC setup..."

kubectl apply -f rbac/namespace.yaml
kubectl apply -f rbac/role.yaml
kubectl apply -f rbac/rolebinding-group.yaml
kubectl apply -f rbac/clusterrole.yaml
kubectl apply -f rbac/clusterrolebinding.yaml

echo "✅ RBAC setup applied successfully!"
Make it executable:

bash
Copy code
chmod +x scripts/apply-rbac.sh
Then run:

bash
Copy code
./scripts/apply-rbac.sh

🧠 6️⃣ Test Access as IAM User
Use the IAM user profile to connect to the cluster:

bash
Copy code
AWS_PROFILE=dev-rahul aws eks update-kubeconfig \
  --name <CLUSTER_NAME> \
  --region <REGION> \
  --alias dev-rahul-context
Switch context:

bash
Copy code
kubectl config use-context dev-rahul-context
Test access:

bash
Copy code
kubectl auth can-i list pods -n rbac-demo
kubectl auth can-i create pods -n rbac-demo
kubectl auth can-i get namespaces
✅ Expected:

Allowed: get/list/watch pods, services, namespaces

Denied: create/update/delete pods

🧹 7️⃣ Cleanup
bash
Copy code
kubectl delete -f rbac/clusterrolebinding.yaml
kubectl delete -f rbac/clusterrole.yaml
kubectl delete -f rbac/rolebinding-group.yaml
kubectl delete -f rbac/role.yaml
kubectl delete ns rbac-demo

🧰 Notes & Best Practices
Prefer mapping IAM roles instead of IAM users for teams.

Use groups (devs, ops, qa) instead of individual user bindings.

Avoid using system:masters group except for cluster admins.

Validate access anytime using:

bash
Copy code
kubectl auth can-i <verb> <resource> -n <namespace>
Manage aws-auth ConfigMap using IaC tools like eksctl, Terraform, or Helm to prevent drift.

🧭 Git Commands to Push Repo
bash
Copy code
git init
git add .
git commit -m "Add complete AWS IAM to Kubernetes RBAC setup"
git branch -M main
git remote add origin https://github.com/<your-username>/k8s-rbac-iam.git
git push -u origin main
🧩 Summary
Layer	Purpose	Scope
IAM Policy	Grants EKS API access	AWS
aws-auth ConfigMap	Maps IAM → Kubernetes user/group	EKS
Role / RoleBinding	Grants namespace-level access	Kubernetes
ClusterRole / ClusterRoleBinding	Grants cluster-wide access	Kubernetes

✅ With this setup, your IAM user has secure, granular, and auditable Kubernetes permissions — ideal for production-grade environments.

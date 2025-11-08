# 🧩 Kubernetes RBAC Complete Setup (from Scratch)

This repository contains a **complete, minimal, and production-ready RBAC setup** for Kubernetes (works on EKS, AKS, GKE, or Minikube).

---

## 🎯 Purpose
To understand and implement Role-Based Access Control (RBAC) in Kubernetes step by step:
- Create namespaces and service accounts
- Define Roles & RoleBindings (namespace-level)
- Define ClusterRoles & ClusterRoleBindings (cluster-level)
- Test access using a sample pod and token

---

## 📘 Folder Structure

```
rbac/
├── namespace.yaml
├── serviceaccount.yaml
├── role.yaml
├── rolebinding.yaml
├── clusterrole.yaml
├── clusterrolebinding.yaml
└── test-pod.yaml

scripts/
└── apply.sh
```



## 🚀 Step-by-Step Setup

 1️⃣ Create Namespace
kubectl apply -f rbac/namespace.yaml

2️⃣ Create Service Account
kubectl apply -f rbac/serviceaccount.yaml

3️⃣ Apply Role & RoleBinding
kubectl apply -f rbac/role.yaml
kubectl apply -f rbac/rolebinding.yaml

4️⃣ Apply ClusterRole & ClusterRoleBinding
kubectl apply -f rbac/clusterrole.yaml
kubectl apply -f rbac/clusterrolebinding.yaml

5️⃣ Verify
kubectl auth can-i list pods --as=system:serviceaccount:rbac-demo:rbac-user

6️⃣ Run Test Pod
kubectl apply -f rbac/test-pod.yaml

🧠 Notes

Namespaced Roles control access within a specific namespace.

ClusterRoles control access across the cluster.

Use ServiceAccounts for pods instead of embedding kubeconfig inside containers.

For real users (developers), use kubectl config set-credentials with OIDC or certificates.

🧰 Cleanup
kubectl delete ns rbac-demo
kubectl delete clusterrole developer-read-only
kubectl delete clusterrolebinding developer-read-only-binding



## 📄 RBAC YAML Files

1️⃣ namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rbac-demo

2️⃣ serviceaccount.yaml
yaml
Copy code
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbac-user
  namespace: rbac-demo

3️⃣ role.yaml (Namespace-level Role)
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

4️⃣ rolebinding.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: rbac-demo
subjects:
- kind: ServiceAccount
  name: rbac-user
  namespace: rbac-demo
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

5️⃣ clusterrole.yaml (Cluster-wide Role)
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

6️⃣ clusterrolebinding.yaml
yaml
Copy code
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-read-only-binding
subjects:
- kind: ServiceAccount
  name: rbac-user
  namespace: rbac-demo
roleRef:
  kind: ClusterRole
  name: developer-read-only
  apiGroup: rbac.authorization.k8s.io

7️⃣ test-pod.yaml (Use this ServiceAccount)
yaml
Copy code
apiVersion: v1
kind: Pod
metadata:
  name: test-rbac-pod
  namespace: rbac-demo
spec:
  serviceAccountName: rbac-user
  containers:
  - name: curl
    image: curlimages/curl
    command: ["sleep", "3600"] ```

You can kubectl exec -it test-rbac-pod -n rbac-demo -- sh and then run:

bash
Copy code
kubectl get pods -n rbac-demo
to test what it can and can’t access.

⚙️ scripts/apply.sh
bash
Copy code
#!/bin/bash
set -e

echo "Applying Kubernetes RBAC Setup..."

kubectl apply -f rbac/namespace.yaml
kubectl apply -f rbac/serviceaccount.yaml
kubectl apply -f rbac/role.yaml
kubectl apply -f rbac/rolebinding.yaml
kubectl apply -f rbac/clusterrole.yaml
kubectl apply -f rbac/clusterrolebinding.yaml
kubectl apply -f rbac/test-pod.yaml

echo "✅ RBAC setup applied successfully!"
Make it executable:

bash
Copy code
chmod +x scripts/apply.sh
🧭 GitHub Upload Instructions
Initialize Git:

bash
Copy code
git init
git add .
git commit -m "Complete RBAC setup from scratch"
Create a new GitHub repo (e.g., k8s-rbac-setup).

Connect and push:

bash
Copy code
git remote add origin https://github.com/<your-username>/k8s-rbac-setup.git
git branch -M main
git push -u origin main

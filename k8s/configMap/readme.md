# 🧩 Kubernetes ConfigMap — Complete Setup (Basic to Advanced)

This repository explains **everything about Kubernetes ConfigMaps** — from scratch to production level — in **simple and practical language**.

---

## 🎯 What Is a ConfigMap?

A **ConfigMap** in Kubernetes is used to store **non-sensitive configuration data** (key-value pairs) separately from your application code.

This helps you:
- Change configurations without rebuilding Docker images  
- Use the same image for **dev**, **staging**, and **production**
- Keep the code clean and environment-independent

---

## 🧱 Why ConfigMaps Are Important for DevOps Engineers

As a DevOps Engineer, ConfigMaps are used daily to:
- Inject environment variables into containers  
- Mount application config files inside pods  
- Manage environment-wise configurations (dev/stage/prod)  
- Integrate with CI/CD pipelines for automated deployments  
- Support **GitOps**, **Helm**, and **Kustomize** workflows  

In short — ConfigMaps help make apps **configurable**, **portable**, and **maintainable**.

---

## ⚙️ 1. Different Ways to Create a ConfigMap

| Method | Description | Example Command |
|--------|--------------|----------------|
| **From literal values** | Key-value directly in CLI | `kubectl create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=debug` |
| **From a file** | Reads key-values from a file | `kubectl create configmap app-config --from-file=config.properties` |
| **From a directory** | Adds all files as keys | `kubectl create configmap app-config --from-file=./configs/` |
| **From YAML** | Declarative method (recommended) | `kubectl apply -f configmap.yaml` |

---

## 🧩 2. Basic ConfigMap Example (YAML)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "mysql-service.demo.svc.cluster.local"
  DB_PORT: "3306"

Apply it:
kubectl apply -f configmap.yaml

🚀 3. How to Use ConfigMap in Pods

You can use ConfigMaps in three ways:

🧩 Method 1 — As Environment Variables

apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: web
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config

✅ Each key in ConfigMap becomes an environment variable in the container.

🧩 Method 2 — As Volume Mount

apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config

✅ Each key in ConfigMap becomes a file under /etc/config.

🧩 Method 3 — Use Specific Keys

apiVersion: v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
✅ Useful when you need only 1 or 2 specific values.

🧠 4. Folder Structure (Best Practice)

k8s/
├── base/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── service.yaml
├── overlays/
│   ├── dev/configmap.yaml
│   ├── staging/configmap.yaml
│   └── prod/configmap.yaml
```
Each environment (dev/staging/prod) has its own ConfigMap file — with the same keys but different values.

🧰 5. Example — Environment Specific Configs
```
base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  API_URL: "http://backend.default.svc.cluster.local"


overlays/prod/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "warn"
  API_URL: "https://api.prod.myapp.com"
 ```
 
🧩 6. ConfigMap with Helm (Dynamic Templates)
```
templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  APP_ENV: {{ .Values.env }}
  LOG_LEVEL: {{ .Values.logLevel }}
  DB_HOST: {{ .Values.db.host }}

  
values.yaml
env: "production"
logLevel: "info"
db:
  host: "db.prod.internal"
  
✅ Helm helps manage ConfigMaps dynamically per environment.
```
🔐 7. ConfigMap vs Secret
```
Feature	ConfigMap	Secret
Data Type	Non-sensitive	Sensitive (passwords, tokens)
Encoding	Plain text	Base64 encoded
Use For	App settings	Credentials, keys
Command	kubectl create configmap	kubectl create secret

👉 Best Practice:
Use ConfigMap for configs (e.g., app mode, URLs)
Use Secrets or External Secrets Operator (ESO) for credentials.
```
⚙️ 8. Real-World DevOps Usage
```
In production setups, DevOps teams:

Store ConfigMaps in Git (for GitOps)

Use Helm/Kustomize to apply environment-specific configs

Use External Secrets Operator for secrets

Automatically reload apps when configs change

Manage everything via CI/CD pipelines (Jenkins, ArgoCD, or GitHub Actions)
```
🪄 9. Advanced Setup — Auto Reload Configs
```
If you want pods to automatically reload when ConfigMaps change:

Install Reloader (by Stakater):
kubectl apply -f https://github.com/stakater/Reloader/releases/latest/download/reloader.yaml


Add label to your Deployment:
metadata:
  labels:
    reloader.stakater.com/auto: "true"
    
✅ When you update the ConfigMap, pods restart automatically.
```
🧩 10. Common Commands
Task	Command
Create ConfigMap
kubectl create configmap app-config --from-literal=ENV=dev
Apply from file	kubectl apply -f configmap.yaml
View data	kubectl get cm app-config -o yaml
Edit	kubectl edit cm app-config
Delete	kubectl delete cm app-config
Restart pods after update	kubectl rollout restart deployment <deployment-name>

🧠 11. Troubleshooting Tips
```
Problem	Cause	Solution
App not reading values	Wrong key name	Verify key matches app code
ConfigMap not found	Wrong namespace	Use -n <namespace> flag
Values not updated	Pod cache old data	Run kubectl rollout restart
Mount path missing	Incorrect mountPath	Check container spec
```
🧾 12. Complete Example Setup
```
deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nginx
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: app-config
            
service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    
configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  WELCOME_MSG: "Hello from ConfigMap!"

  
Apply all:
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

🧰 13. Best Practices Summary
✅ Store configmaps in Git
✅ Separate configs by environment
✅ Never store passwords here
✅ Use meaningful key names
✅ Add labels for tracking
✅ Restart pods or use reloader for updates
✅ Use Helm or Kustomize for automation

🎉 14. Cleanup Commands
```
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
kubectl delete -f configmap.yaml
```
🧩 Summary
Level	Focus	Example
Basic	Create & use configmaps	Inject ENV vars
Intermediate	Environment-wise configs	Helm / Kustomize
Advanced	Auto reload + GitOps	Reloader + CI/CD


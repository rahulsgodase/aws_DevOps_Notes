# HAProxy Load Balancing for Docker Apps on EC2 (Without ALB)

This guide explains how to deploy a Dockerized application across multiple EC2 instances and load-balance traffic using **HAProxy**, without using an AWS Application Load Balancer (ALB).

---

## 🔷 High-Level Idea (What You Are Building)

- 5 EC2 instances running the same application in Docker
- 1 EC2 instance running HAProxy
- HAProxy receives client traffic and distributes it to all application instances

---

## 🔷 What HAProxy Does Here

HAProxy works as:
- A **Reverse Proxy**
- A **Layer 4 / Layer 7 Load Balancer**
- A **manual but powerful alternative to AWS ALB**


::contentReference[oaicite:1]{index=1}


---

## 🔷 Architecture Overview (Logical View)

Client (Browser / API Client)
|
| HTTP (80) / HTTPS (443)
|
HAProxy (EC2)
|
|---- App (Docker) EC2-1 : 8080
|---- App (Docker) EC2-2 : 8080
|---- App (Docker) EC2-3 : 8080
|---- App (Docker) EC2-4 : 8080
|---- App (Docker) EC2-5 : 8080



---

## 🔷 Infrastructure Summary

| Component         | Count | Purpose                          |
|------------------|-------|----------------------------------|
| EC2 (App Nodes)  | 5     | Run Dockerized application       |
| EC2 (HAProxy)    | 1     | Load balancer / reverse proxy    |
| Docker           | Yes   | Container runtime                |
| HAProxy          | Yes   | Traffic distribution             |
| ALB              | No    | Replaced by HAProxy              |


## 🔷 Step-by-Step Implementation

---

### Step 1️⃣ Run Application in Docker on All EC2 Instances

```bash
docker run -d \
  --name myapp \
  -p 8080:8080 \
  myapp:latest

Step 2️⃣ Choose HAProxy Node

Options:

Use one of the existing EC2 instances

Use a separate EC2 instance (recommended for production)

Step 3️⃣ Install HAProxy
sudo apt update
sudo apt install haproxy -y

🔷 Step 4️⃣ Configure HAProxy (HTTP + HTTPS)
sudo vi /etc/haproxy/haproxy.cfg
✅ Production-Ready Configuration
global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend http_front
    bind *:80
    redirect scheme https if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/app.pem
    default_backend app_servers

backend app_servers
    balance roundrobin
    option httpchk GET /

    server app1 10.0.1.10:8080 check
    server app2 10.0.1.11:8080 check
    server app3 10.0.1.12:8080 check
    server app4 10.0.1.13:8080 check
    server app5 10.0.1.14:8080 check
✔️ Always use private IPs of EC2 instances.

🔷 Step 5️⃣ Restart HAProxy
sudo systemctl restart haproxy
sudo systemctl status haproxy

Access:

http://<haproxy-public-ip>
https://<haproxy-public-ip>

🔷 Step 6️⃣ Health Checks

HAProxy:

Continuously checks /

Removes unhealthy backends automatically

Re-adds healthy backends

🔷 SSL Certificate Creation
Option 1️⃣ Self-Signed Certificate (Testing Only)
sudo mkdir -p /etc/haproxy/certs
cd /etc/haproxy/certs

openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout app.key \
-out app.crt
Combine certificate and key:
cat app.crt app.key > app.pem
Option 2️⃣ Let’s Encrypt Certificate (Production)
sudo apt install certbot -y
sudo certbot certonly --standalone -d yourdomain.com

cat fullchain.pem privkey.pem > app.pem


Move to:

/etc/haproxy/certs/app.pem

🔷 Required Ports & Security Groups
Component	Port	Purpose
HAProxy	80	HTTP
HAProxy	443	HTTPS
HAProxy	8404	Stats Dashboard
App EC2	8080	Application
🔷 Optional: HAProxy Stats Dashboard
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s


Access:

http://<haproxy-ip>:8404/stats

🔷 HAProxy vs ALB
Feature	HAProxy	ALB
Cost	Very Low	Higher
Control	Full	Limited
TLS	Manual	Managed
Auto Scaling	Manual	Automatic
Vendor Lock-in	No	Yes
🔷 When to Use HAProxy

Full infrastructure control

Cost-sensitive environments

Cloud-agnostic setup

🔷 When NOT to Use HAProxy

Need managed TLS + WAF

Zero maintenance

Built-in autoscaling

🔷 Interview-Ready Summary

“Instead of using an ALB, we deployed HAProxy as a reverse proxy to load-balance traffic acrossDockerized
applications running on multiple EC2 instances using health checks and round-robin strategy.”

✅ Conclusion

HAProxy is a production-ready, cost-effective, and powerful alternative to AWS ALB when infrastructure is self-managed.

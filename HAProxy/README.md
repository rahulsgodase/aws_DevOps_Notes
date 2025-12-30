# HAProxy Load Balancing for Docker Apps on EC2 (Without ALB)

This guide explains how to deploy a Dockerized application across multiple EC2 instances and load-balance traffic using **HAProxy**, without using an AWS Application Load Balancer (ALB).

---

## 🔷 High-Level Idea (What You Are Building)

- 5 EC2 instances running the same application in Docker
- 1 EC2 instance running HAProxy
- HAProxy receives client traffic and distributes it to all application instances


::contentReference[oaicite:0]{index=0}


---

## 🔷 What HAProxy Does Here

HAProxy works as:
- A **Reverse Proxy**
- A **Layer 4 / Layer 7 Load Balancer**
- A **manual but powerful alternative to AWS ALB**


::contentReference[oaicite:1]{index=1}


---

## 🔷 Architecture Overview (Logical View)

Client
|
|
HAProxy (EC2 Instance)
|
|---- App (Docker) on EC2-1 : 8080
|---- App (Docker) on EC2-2 : 8080
|---- App (Docker) on EC2-3 : 8080
|---- App (Docker) on EC2-4 : 8080
|---- App (Docker) on EC2-5 : 8080

yaml
Copy code


::contentReference[oaicite:2]{index=2}


---

## 🔷 Step-by-Step Implementation

### Step 1️⃣ Run the Application in Docker on All EC2 Instances

On **each EC2 instance**, run:

```bash
docker run -d \
  --name myapp \
  -p 8080:8080 \
  myapp:latest
Each instance will now serve the application at:

cpp
Copy code
http://<instance-private-ip>:8080




Step 2️⃣ Choose the HAProxy Node
You can:

Use one of the 5 EC2 instances, or

Use a separate 6th EC2 instance (recommended for production)





Step 3️⃣ Install HAProxy
bash
Copy code
sudo apt update
sudo apt install haproxy -y




Step 4️⃣ Configure HAProxy
Edit the configuration file:

bash
Copy code
sudo vi /etc/haproxy/haproxy.cfg
Basic HTTP Configuration
cfg
Copy code
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
    default_backend app_servers

backend app_servers
    balance roundrobin
    option httpchk GET /

    server app1 10.0.1.10:8080 check
    server app2 10.0.1.11:8080 check
    server app3 10.0.1.12:8080 check
    server app4 10.0.1.13:8080 check
    server app5 10.0.1.14:8080 check
✅ Always use private IPs of EC2 instances.





Step 5️⃣ Restart HAProxy
bash
Copy code
sudo systemctl restart haproxy
sudo systemctl status haproxy
Access the application:

cpp
Copy code
http://<haproxy-public-ip>




Step 6️⃣ Health Checks (Important)
HAProxy:

Continuously checks the / endpoint

Removes unhealthy backends automatically

Re-adds them when healthy again

This provides basic self-healing.





🔷 Optional: HAProxy Stats Dashboard
Add this to haproxy.cfg:

cfg
Copy code
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
Access:

arduino
Copy code
http://<haproxy-ip>:8404/stats




🔷 HAProxy vs ALB (Comparison)
Feature	HAProxy	ALB
Cost	Very Low	Higher
Control	Full	Limited
Setup	Manual	Easy
Auto Scaling	Manual	Automatic
TLS	Manual	Managed
WAF	No	Yes

🔷 When to Use HAProxy
You want full control

You understand infrastructure

You want low cost

You want cloud-agnostic setup

🔷 When NOT to Use HAProxy
You need managed TLS & WAF

You want zero maintenance

You need built-in autoscaling

🔷 Interview-Ready Summary
“Instead of using an ALB, we can deploy HAProxy as a reverse proxy. It distributes traffic to Dockerized applications running on multiple EC2 instances using health checks and round-robin or least-connection strategies.”

✅ Conclusion
HAProxy is a production-ready, cost-effective, and powerful load-balancing solution when you are comfortable managing infrastructure manually.

yaml
Copy code

---



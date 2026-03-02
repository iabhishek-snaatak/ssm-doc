Envoy Proxy Step-by-Step Guide (Corrected & Production-Ready)

This guide explains Envoy Proxy from installation to routing traffic with clear and practical examples.

1. What is Envoy Proxy?

Envoy is a high-performance proxy server.

It sits between:

Client → Envoy → Backend Server

Example flow:

Browser → Envoy → nginx
curl → Envoy → OpenSearch
App → Envoy → API server

Envoy receives requests and forwards them to backend servers.

Envoy is used for:

• Load balancing
• Routing traffic
• TLS termination (HTTPS handling)
• Security
• Service mesh (Istio uses Envoy internally)
• Microservices communication

Envoy works at:

Layer 4 → TCP
Layer 7 → HTTP / HTTPS

2. Core Envoy Concepts

These four concepts are the foundation of Envoy.

Downstream

Downstream = Client connecting to Envoy

Example:

curl https://opensearch.opstree.dev

Client → Envoy

Client is downstream.

Upstream

Upstream = Backend server Envoy connects to

Example:

Envoy → 192.168.8.188:9200

Backend server is upstream.

Listener

Listener defines:

• Which port Envoy listens on
• Which protocol (HTTP / HTTPS)

Example:

Envoy listening on port 443
Cluster

Cluster defines backend servers.

Example:

Cluster name: open_search
Backend: 192.168.8.188:9200
Route

Route defines:

Which request goes to which cluster.

Example:

Domain: opensearch.opstree.dev
Route → open_search cluster
3. Install Envoy (Ubuntu)

Run:

wget -O- https://apt.envoyproxy.io/signing.key | sudo gpg --dearmor -o /etc/apt/keyrings/envoy-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/envoy-keyring.gpg] https://apt.envoyproxy.io jammy main" | sudo tee /etc/apt/sources.list.d/envoy.list

sudo apt-get update

sudo apt-get install envoy

envoy --version

Verify:

envoy --version
4. Envoy Configuration Location

Envoy does not use a default config automatically.

You must specify config file manually.

Common location:

/etc/envoy/envoy.yaml

Run using:

envoy -c /etc/envoy/envoy.yaml
5. Understanding Your Configuration

Your config has 2 listeners:

Listener 1 → Port 80 → Redirect HTTP to HTTPS
Listener 2 → Port 443 → Handle HTTPS traffic

Flow:

HTTP request → port 80 → redirect → HTTPS port 443 → backend cluster

Listener: HTTP Redirect
port_value: 80

Purpose:

Redirect all HTTP traffic to HTTPS.

Example:

http://opensearch.opstree.dev

Redirects to:

https://opensearch.opstree.dev
Listener: HTTPS Traffic
port_value: 443

Purpose:

Accept HTTPS traffic and route to backend clusters.

Uses TLS certificate:

/etc/envoy/tls/tls.crt
/etc/envoy/tls/tls.key

Envoy decrypts HTTPS and forwards request to backend.

This is called:

TLS termination

6. Clusters (Backend Servers)

Your config defines 3 clusters:

Cluster 1: pmm_cluster

Backend:

192.168.1.1:5601
Cluster 2: open_search

Backend:

192.168.8.188:5601
Cluster 3: open_search_metrics

Backend:

192.168.8.188:9200

Envoy forwards traffic to these servers.

7. Routing Logic

Routing is based on domain and path.

Example 1:

Request:

https://pmm.opstree.dev/

Envoy → pmm_cluster

Example 2:

Request:

https://opensearch.opstree.dev/

Envoy → open_search cluster

Example 3:

Request:

https://opensearch.opstree.dev/metrics

Envoy → open_search_metrics cluster

Example 4:

Any unknown domain:

Envoy → pmm_cluster (default)

8. Start Envoy

Run:

envoy -c /etc/envoy/envoy.yaml

Envoy will start listening on:

Port 80
Port 443
Port 9901 (admin)
9. Verify Envoy Running

Check process:

ps aux | grep envoy

Check admin interface:

http://localhost:9901

This shows:

• Clusters
• Listeners
• Stats
• Health

10. Test Envoy

Test HTTPS routing:

curl -k https://opensearch.opstree.dev

Flow:

Client → Envoy:443 → open_search cluster → Backend server

Test metrics route:

curl -k https://opensearch.opstree.dev/metrics

Flow:

Client → Envoy → open_search_metrics → Backend

11. Run Envoy in Background

Foreground:

envoy -c /etc/envoy/envoy.yaml

Background:

envoy -c /etc/envoy/envoy.yaml -l info &

Production recommended:

Use systemd service.

12. Production Architecture Example

Real world architecture:

Internet
   │
   ▼
Envoy Proxy Server
   │
   ├── OpenSearch Server
   ├── PMM Server
   └── API Servers

Envoy handles:

• Routing
• Load balancing
• TLS termination
• Health checking

13. Envoy Admin Interface

Configured here:

port_value: 9901

Access:

http://localhost:9901

Useful endpoints:

/stats
/clusters
/listeners

This is extremely useful for debugging.

14. Stop Envoy

Stop process:

pkill envoy

Or:

kill -9 <PID>
15. Correct Flow Summary (Your Config)

Actual flow based on your YAML:

Client
   │
   ▼
Envoy port 80 → Redirect to HTTPS
   │
   ▼
Envoy port 443 (TLS termination)
   │
   ▼
Route based on domain/path
   │
   ├── pmm_cluster → 192.168.1.1:5601
   ├── open_search → 192.168.8.188:5601
   └── open_search_metrics → 192.168.8.188:9200
Critical Fixes from Your Original Doc

Your original doc had these incorrect parts:

Wrong:

Envoy listening on port 8080

Correct:

Envoy listening on port 80 and 443

Wrong:

curl https://envoy.opstree.dev:808

Correct:

curl https://envoy.opstree.dev

because config uses port 443.

Envoy is essentially a programmable traffic brain. It reads a YAML file and becomes whatever network creature you describe: load balancer, TLS terminator, API gateway, or service mesh sidecar. Same binary. Different personality. Pure software alchemy.

The next evolutionary leap is understanding dynamic configuration (xDS) and how Istio uses Envoy as distributed nervous tissue across thousands of services.

isko .md mei do
Analyzed
# This script creates a Markdown (.md) file of the corrected Envoy Proxy guide
# using pypandoc as required by the file generation rules.

import pypandoc

content = """
# Envoy Proxy Step-by-Step Guide (Corrected & Production-Ready)

This guide explains Envoy Proxy from installation to routing traffic with clear and practical examples.

---

# 1. What is Envoy Proxy?

Envoy is a high-performance proxy server.

It sits between:

Client → Envoy → Backend Server

Example flow:

Browser → Envoy → nginx  
curl → Envoy → OpenSearch  
App → Envoy → API server  

Envoy receives requests and forwards them to backend servers.

Envoy is used for:

- Load balancing  
- Routing traffic  
- TLS termination (HTTPS handling)  
- Security  
- Service mesh (Istio uses Envoy internally)  
- Microservices communication  

Envoy works at:

- Layer 4 → TCP  
- Layer 7 → HTTP / HTTPS  

---

# 2. Core Envoy Concepts

## Downstream

Downstream = Client connecting to Envoy

Example:

curl https://opensearch.opstree.dev

Client → Envoy

---

## Upstream

Upstream = Backend server Envoy connects to

Example:

Envoy → 192.168.8.188:9200

---

## Listener

Listener defines:

- Which port Envoy listens on  
- Which protocol (HTTP / HTTPS)

Example:

Envoy listening on port 443

---

## Cluster

Cluster defines backend servers.

Example:

Cluster name: open_search  
Backend: 192.168.8.188:9200

---

## Route

Route defines which request goes to which cluster.

Example:

Domain: opensearch.opstree.dev  
Route → open_search cluster

---

# 3. Install Envoy (Ubuntu)


wget -O- https://apt.envoyproxy.io/signing.key
 | sudo gpg --dearmor -o /etc/apt/keyrings/envoy-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/envoy-keyring.gpg] https://apt.envoyproxy.io
 jammy main" | sudo tee /etc/apt/sources.list.d/envoy.list

sudo apt-get update

sudo apt-get install envoy

envoy --version


---

# 4. Envoy Configuration Location

Common location:


/etc/envoy/envoy.yaml


Run:


envoy -c /etc/envoy/envoy.yaml


---

# 5. Listener Overview

Listener 1 → Port 80 → Redirect HTTP to HTTPS  
Listener 2 → Port 443 → Handle HTTPS traffic  

---

# 6. Clusters

pmm_cluster → 192.168.1.1:5601  
open_search → 192.168.8.188:5601  
open_search_metrics → 192.168.8.188:9200  

---

# 7. Routing Logic

https://pmm.opstree.dev → pmm_cluster  

https://opensearch.opstree.dev → open_search  

https://opensearch.opstree.dev/metrics → open_search_metrics  

Unknown domain → default cluster

---

# 8. Start Envoy


envoy -c /etc/envoy/envoy.yaml


Envoy listens on:

- Port 80  
- Port 443  
- Port 9901 (admin)

---

# 9. Verify Envoy


ps aux | grep envoy


Admin panel:

http://localhost:9901

---

# 10. Test


curl -k https://opensearch.opstree.dev


Flow:

Client → Envoy → Cluster → Backend

---

# 11. Run in Background


envoy -c /etc/envoy/envoy.yaml -l info &


---

# 12. Architecture

Internet  
↓  
Envoy  
↓  
Backend servers  

---

# 13. Admin Interface

http://localhost:9901

Endpoints:

/stats  
/clusters  
/listeners  

---

# 14. Stop Envoy


pkill envoy


---

# 15. Final Flow

Client  
↓  
Envoy:80 → redirect → HTTPS  
↓  
Envoy:443  
↓  
Route  
↓  
Cluster  
↓  
Backend  

---

Envoy is a programmable proxy that controls traffic routing, security, and load balancing.
"""

output_file = "/mnt/data/envoy-proxy-guide.md"

pypandoc.convert_text(
    content,
    'md',
    format='md',
    outputfile=output_file,
    extra_args=['--standalone']
)

output_file
Result
'/mnt/data/envoy-proxy-guide.md'

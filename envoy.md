# Envoy Proxy Step‑by‑Step Guide (Simple English)

This guide explains Envoy Proxy from installation to running and routing traffic. It is written in simple English and practical steps.

---

# 1. What is Envoy Proxy?

Envoy is a high‑performance proxy server.

It sits between:

* Client (browser, curl, app)
* Backend server (nginx, nodejs, java, etc.)

Flow:

Client → Envoy → Backend Server

Envoy receives request and forwards it to correct backend server.

Envoy is used for:

* Load balancing
* Routing
* Security
* Service mesh (Istio, etc.)
* Microservices

Envoy works on:

* Layer 4 (TCP)
* Layer 7 (HTTP)

---

# 2. Important Envoy Concepts

## Downstream

Client → Envoy

Client sending request to Envoy.

Example:

curl [http://localhost:8080](http://localhost:8080)

Here client is downstream.

## Upstream

Envoy → Backend server

Example:

Envoy → nginx (localhost:9000)

nginx is upstream.

## Listener

Listener defines:

* Which port Envoy listens on

Example:

Envoy listens on port 8080

## Cluster

Cluster defines:

* Backend servers

Example:

nginx running on port 9000

## Route

Route defines:

* Which request goes to which cluster

Example:

/ → nginx cluster

---

# 3. Installation (Ubuntu Linux)

Step 1: Add Envoy repository key

```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://apt.envoyproxy.io/signing.key | sudo gpg --dearmor -o /etc/apt/keyrings/envoy-keyring.gpg
```

Step 2: Add repository

```
echo "deb [signed-by=/etc/apt/keyrings/envoy-keyring.gpg] https://apt.envoyproxy.io jammy main" | sudo tee /etc/apt/sources.list.d/envoy.list
```

Step 3: Update

```
sudo apt update
```

Step 4: Install Envoy

```
sudo apt install envoy -y
```

Step 5: Verify

```
envoy --version
```

---

# 4. Create Backend Server

Install nginx:

```
sudo apt install nginx -y
```

Change nginx port:

```
sudo nano /etc/nginx/sites-available/default
```

Change:

```
listen 80;
```

to

```
listen 9000;
```

Restart nginx:

```
sudo systemctl restart nginx
```

Test:

```
curl http://localhost:9000
```

nginx is now backend server.

---

# 5. Create Envoy Configuration File

Create file:

```
nano envoy.yaml
```

Paste:

```
static_resources:
  listeners:
    - name: listener_http
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080

      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager

                stat_prefix: ingress_http

                route_config:
                  name: local_route

                  virtual_hosts:
                    - name: backend
                      domains: ["*"]

                      routes:
                        - match:
                            prefix: "/"

                          route:
                            cluster: nginx_cluster

                http_filters:
                  - name: envoy.filters.http.router

  clusters:
    - name: nginx_cluster

      connect_timeout: 5s

      type: logical_dns

      lb_policy: round_robin

      load_assignment:
        cluster_name: nginx_cluster

        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 127.0.0.1
                      port_value: 9000
```

Save file.

---

# 6. Start Envoy

Run command:

```
envoy -c envoy.yaml --log-level info
```

Envoy will start.

Envoy now listening on port:

8080

---

# 7. Test Envoy

Run:

```
curl http://localhost:8080
```

Flow:

curl → Envoy:8080 → nginx:9000

Envoy forwards request to nginx.

---

# 8. How Routing Works (Simple Explanation)

Example request:

Client:

[http://localhost:8080](http://localhost:8080)

Step 1:

Envoy listener receives request

Step 2:

Envoy checks route

prefix: /

Step 3:

Envoy selects cluster

nginx_cluster

Step 4:

Envoy forwards request to

127.0.0.1:9000

Step 5:

nginx responds

Step 6:

Envoy sends response back to client

---

# 9. Which Server Envoy Runs On?

Envoy runs on proxy server.

Example architecture:

Client → Envoy Server → Backend Server

Example real world:

Server 1:
Envoy running

Server 2:
nginx running

Server 3:
nodejs running

Envoy routes traffic to correct server.

---

# 10. Example Multiple Backend Routing

Example:

nginx1 → port 9000
nginx2 → port 9001

Envoy will load balance.

Envoy automatically distributes traffic.

---

# 11. How to Run Envoy in Background

```
envoy -c envoy.yaml -d
```

or use systemd service.

---

# 12. Production Architecture

Example:

Internet
↓
Envoy
↓
Multiple backend servers

Envoy handles:

* Load balancing
* Routing
* Health checks
* Security

---

# 13. Real World Example

Example:

User opens:

[www.example.com](http://www.example.com)

Flow:

User → Envoy → API server
User → Envoy → Auth server
User → Envoy → Web server

Envoy decides where to send request.

---

# 14. Important Commands

Start Envoy:

```
envoy -c envoy.yaml
```

Check process:

```
ps aux | grep envoy
```

Stop Envoy:

```
pkill envoy
```

---

# 15. Summary

Envoy is proxy server.

It:

Receives traffic
Routes traffic
Load balances traffic

Envoy sits between client and backend.

Client never talks directly to backend.

Envoy controls everything.

---

# End of Guide

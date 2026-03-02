# Envoy Proxy Step‑by‑Step Guide 

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


```
wget -O- https://apt.envoyproxy.io/signing.key | sudo gpg --dearmor -o /etc/apt/keyrings/envoy-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/envoy-keyring.gpg] https://apt.envoyproxy.io jammy main" | sudo tee /etc/apt/sources.list.d/envoy.list
sudo apt-get update
sudo apt-get install envoy
envoy --version

```

<img width="1848" height="1054" alt="image" src="https://github.com/user-attachments/assets/757be3a3-b96d-4575-8e37-eb53f232cdaf" />


---


# 4. Create Envoy Configuration 

Envoy does not have a single hard-coded default configuration file location. It only loads a configuration file when you explicitly provide the path using the -c flag.

However, when Envoy is installed using standard Linux packages (like apt on Ubuntu/Debian), the commonly used default location is:

```
/etc/envoy/envoy.yaml
```

## configuration file

```
static_resources:

  listeners:


    - name: listener_http_redirect
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 80

      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager

                stat_prefix: ingress_http_redirect

                route_config:
                  virtual_hosts:
                    - name: redirect_all
                      domains: ["*"]
                      routes:
                        - match:
                            prefix: "/"
                          redirect:
                            https_redirect: true

                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router


    - name: listener_https
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 443

      filter_chains:

        - transport_socket:
            name: envoy.transport_sockets.tls
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext

              common_tls_context:
                tls_certificates:
                  - certificate_chain:
                      filename: /etc/envoy/tls/tls.crt
                    private_key:
                      filename: /etc/envoy/tls/tls.key

          filters:

            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager

                stat_prefix: ingress_https

                route_config:
                  name: route_config

                  virtual_hosts:

                    - name: pmm_service
                      domains: ["pmm.opstree.dev"]

                      routes:

                        - match:
                            prefix: "/"
                          route:
                            cluster: "pmm_cluster"

                    - name: open_search
                      domains: ["opensearch.opstree.dev"]

                      routes:

                        - match:
                            prefix: "/metrics"
                          route:
                            cluster: "open_search_metrics"
                        - match:
                            prefix: "/"
                          route:
                            cluster: "open_search"

                    - name: default_service
                      domains: ["*"]

                      routes:

                        - match:
                            prefix: "/"
                          route:
                            cluster: "pmm_cluster"


                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router




  clusters:


    - name: pmm_cluster

      connect_timeout: 5s

      type: STATIC

      lb_policy: ROUND_ROBIN


      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext


      dns_lookup_family: V4_ONLY

      health_checks:

        - timeout: 2s
          interval: 5s
          unhealthy_threshold: 3
          healthy_threshold: 2

          http_health_check:
            path: /

      load_assignment:

        cluster_name: pmm_cluster

        endpoints:

          - lb_endpoints:


              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.1.1
                      port_value: 5601



    - name: open_search

      connect_timeout: 5s

      type: STATIC

      lb_policy: ROUND_ROBIN


      dns_lookup_family: V4_ONLY

      health_checks:

        - timeout: 2s
          interval: 5s
          unhealthy_threshold: 3
          healthy_threshold: 2

          http_health_check:
            path: /

      load_assignment:

        cluster_name: open_search

        endpoints:

          - lb_endpoints:


              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.8.188
                      port_value: 5601



    - name: open_search_metrics

      connect_timeout: 5s

      type: STATIC

      lb_policy: ROUND_ROBIN


      dns_lookup_family: V4_ONLY

      health_checks:

        - timeout: 2s
          interval: 5s
          unhealthy_threshold: 3
          healthy_threshold: 2

          http_health_check:
            path: /

      load_assignment:

        cluster_name: open_search_metrics

        endpoints:

          - lb_endpoints:


              - endpoint:
                  address:
                    socket_address:
                      address: 192.168.8.188
                      port_value: 9200





admin:

  access_log_path: /tmp/envoy_admin.log

  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901

```

---

# 6. Start Envoy

manually Run command:

```
envoy -c envoy.yaml 
```

Envoy will start.

Envoy now listening on port:

443

---

# 7. Test Envoy

Run:

```
curl https://envoy.opstree.dev
```

---
## 8. How Routing Works (Simple Explanation)

This section explains how Envoy routes incoming client requests to the correct backend service based on **port, domain, and path**.

---

### Example 1: HTTP to HTTPS Redirect

**Client Request:**

```
http://pmm.opstree.dev
```

**Step 1:**

Client sends HTTP request on port **80**

**Step 2:**

Envoy `listener_http_redirect` receives the request

**Step 3:**

Envoy checks route

```
prefix: /
```

**Step 4:**

Envoy redirects request to HTTPS

```
https://pmm.opstree.dev
```

---

### Example 2: PMM Service Routing

**Client Request:**

```
https://pmm.opstree.dev
```

**Step 1:**

Envoy `listener_https` receives request on port **443**

**Step 2:**

Envoy checks domain

```
pmm.opstree.dev
```

**Step 3:**

Envoy selects cluster

```
pmm_cluster
```

**Step 4:**

Envoy forwards request to backend

```
192.168.1.1:5601
```

**Step 5:**

PMM service sends response

**Step 6:**

Envoy returns response to client

---

### Example 3: OpenSearch Main UI Routing

**Client Request:**

```
https://opensearch.opstree.dev/
```

**Step 1:**

Envoy receives request on port **443**

**Step 2:**

Envoy checks domain

```
opensearch.opstree.dev
```

**Step 3:**

Envoy matches route

```
prefix: /
```

**Step 4:**

Envoy selects cluster

```
open_search
```

**Step 5:**

Envoy forwards request to backend

```
192.168.8.188:5601
```

**Step 6:**

OpenSearch responds

**Step 7:**

Envoy sends response back to client

---

### Example 4: OpenSearch Metrics Routing

**Client Request:**

```
https://opensearch.opstree.dev/metrics
```

**Step 1:**

Envoy receives request

**Step 2:**

Envoy checks domain

```
opensearch.opstree.dev
```

**Step 3:**

Envoy matches route

```
prefix: /metrics
```

**Step 4:**

Envoy selects cluster

```
open_search_metrics
```

**Step 5:**

Envoy forwards request to backend

```
192.168.8.188:9200
```

**Step 6:**

Metrics service responds

**Step 7:**

Envoy sends response to client

---

### Example 5: Default Routing (Unknown Domain)

**Client Request:**

```
https://unknown-domain.com
```

**Step 1:**

Envoy receives request

**Step 2:**

Envoy matches default virtual host

```
domains: *
```

**Step 3:**

Envoy selects default cluster

```
pmm_cluster
```

**Step 4:**

Envoy forwards request

```
192.168.1.1:5601
```

**Step 5:**

Envoy sends response back to client

---

### Summary Flow

```
Client → Listener → Virtual Host → Route → Cluster → Backend → Response → Client
```

Envoy uses:

* **Listener** → accepts incoming connection
* **Virtual Host** → matches domain
* **Route** → matches path
* **Cluster** → defines backend server
* **Endpoint** → actual backend IP and port


Envoy routes traffic to correct server.

---

# 11. How to Run Envoy in Background

```
envoy -c envoy.yaml -d
```

or use systemd service.

---


# End of Guide

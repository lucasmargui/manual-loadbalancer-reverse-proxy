# Nginx Reverse Proxy Setup with Docker

Advanced Nginx reverse proxy architecture for multi-service Docker environments, optimized for maintainability, scalability, and local development.

## Manual - Reverse Proxy + Load Balancer

This section explains how to configure Nginx to act as a **reverse proxy** and **load balancer** for multiple Docker services.

<details><summary> Show details! </summary>

## Table of Contents

- [Overview](#overview)  
- [Architecture](#architecture)  
- [Docker Compose Architecture](#docker-compose-architecture)  
- [Nginx Configuration](#nginx-configuration)  
- [Local Testing with Hosts](#local-domain-simulation)  
- [Request Flow](#request-flow)  
- [Getting Started](#getting-started)  
- [Best Practices & Notes](#best-practices--notes)  

---

## Overview

A **reverse proxy** acts as the gateway for client requests, routing traffic to the appropriate backend service. This setup provides:

- Multi-domain hosting on a single public IP
- Centralized SSL/TLS management
- Load balancing, caching, and security
- Containerized service isolation

### Services in this setup:
- nginx-proxy: central reverse proxy
- main-domain-server, module1-server, module2-server: backend HTTP services (Apache)
- Local hosts simulate domains/subdomains for development/testing

---

## Architecture

```arduino
               +------------------+
               |  nginx-proxy     |
               |  (Reverse Proxy) |  
               +--------+---------+
                        |
    ---------------------------------------------------
    |                     |                           |
+---------------+        +----------------+        +----------------+
| module1       |        | module2        |        | module3        |
| module1-server|        | module2-server |        | module3-server |
+---------------+        +----------------+        +----------------+

```

- All services are Dockerized
- Nginx dynamically routes requests based on Host headers
- Backend containers serve isolated content for each domain/subdomai

--- 

## Docker Compose Architecture

`docker-compose.yml` defines isolated services with volume mounting for content:

```
version: "3.9"

networks:
  webnet:
    driver: bridge

services:
  nginx-proxy:
    image: nginx:latest
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./certificates:/etc/letsencrypt
    depends_on:
      - module1-server
      - module2-server
      - module3-server
    networks:
      - webnet

  
  module1-server-1:
    image: httpd:latest
    container_name: module1-server-1
    restart: always
    volumes:
      - ./example-module1:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module1-server-2:
    image: httpd:latest
    container_name: module1-server-2
    restart: always
    volumes:
      - ./example-module1:/usr/local/apache2/htdocs/
    networks:
      - webnet


  module2-server-1:
    image: httpd:latest
    container_name: module2-server-1
    restart: always
    volumes:
      - ./example-module2:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module2-server-2:
    image: httpd:latest
    container_name: module2-server-2
    restart: always
    volumes:
      - ./example-module2:/usr/local/apache2/htdocs/
    networks:
      - webnet

 
  module3-server-1:
    image: httpd:latest
    container_name: module3-server-1
    restart: always
    volumes:
      - ./example-module3:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module3-server-2:
    image: httpd:latest
    container_name: module3-server-2
    restart: always
    volumes:
      - ./example-module3:/usr/local/apache2/htdocs/
    networks:
      - webnet


```

- depends_on ensures backend services start before Nginx
- Apache serves static content via mounted volumes
- Volumes make local development iterative and fast

--- 

## Nginx Configuration

Reverse proxy configuration is modular: each server block corresponds to a domain/subdomain:

```
# Module 1
upstream module1_backend {
    server module1-server-1:80;
    server module1-server-2:80;
}

server {
    listen 80;
    server_name module1.main-domain-example.online;

    location / {
        proxy_pass http://module1_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Module 2
upstream module2_backend {
    server module2-server-1:80;
    server module2-server-2:80;
}

server {
    listen 80;
    server_name module2.main-domain-example.online;

    location / {
        proxy_pass http://module2_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Module 3
upstream module3_backend {
    server module3-server-1:80;
    server module3-server-2:80;
}

server {
    listen 80;
    server_name module3.main-domain-example.online;

    location / {
        proxy_pass http://module3_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```


- Maintains client IP and protocol headers for logging, security, and analytics-
- Easy to extend with SSL, caching, or custom routing logic
- Modular configuration allows adding new modules without downtime

--- 

## Local Domain Simulation

For development without a public DNS, edit your hosts file:

```bash
# Local test domains and subdomains for Nginx proxy
127.0.0.1 main-domain-example.online
127.0.0.1 module1.main-domain-example.online
127.0.0.1 module2.main-domain-example.online

```
<img width="426" height="106" alt="image" src="https://github.com/user-attachments/assets/28a7aaee-58a1-4e59-9614-d24a50032478" />

- Enables multi-subdomain testing on a single local machine
- Works seamlessly with Docker networking

### Request Flow

1. A client requests http://module1.main-domain-example.online
2. Nginx checks the server_name in its configuration
3. The request is forwarded to module1-main-domain-server container
4. Apache inside the container serves the content
5. Response is sent back through Nginx to the client

This flow can be extended for:
- SSL termination (letsencrypt/certbot)
- Load balancing (round-robin, least connections)
- Caching static assets for performance

--- 

## Getting Started

1. Clone the repository
   ```
    git clone <repo-url>
    cd <repo-directory>
   ```
3. Build and start containers:
   ```
   docker-compose up -d
   ```
4. Edit `/etc/hosts` (or `C:\Windows\System32\drivers\etc\hosts` on Windows) with test domains

  ```bash
  127.0.0.1 main-domain-example.online
  127.0.0.1 module1.main-domain-example.online
  127.0.0.1 module2.main-domain-example.online
  ```

6. Open your browser and access:
   
    - `http://main-domain-example.online`
    - `http://module1.main-domain-example.online`
    - `http://module2.main-domain-example.online`
  
  <img width="1377" height="313" alt="image" src="https://github.com/user-attachments/assets/f1757db5-02cb-4677-b106-03451130ad07" />

--- 
  
## Best Practices & Notes

- Use named Docker networks for isolation and explicit container communication
- Prefer environment variables for dynamic backend URLs
- Version control your Nginx configuration for rollback and audit
- Consider Docker healthchecks for backend containers to ensure Nginx only forwards to healthy services
- For production, combine with HTTPS, rate limiting, and caching strategies

--- 

This architecture is production-ready, highly maintainable, and extensible for microservices, modular applications, or multi-tenant hosting environments.

</details>

---

## Kubernetes like - Reverse Proxy + Load Balancer

This project demonstrates a professional multi-module web setup with load balancing and dynamic reverse proxy using Docker Compose, inspired by Kubernetes orchestration patterns.

<details><summary> Show details! </summary>

It uses:

- Dynamic Nginx Proxy (jwilder/nginx-proxy) to automatically route HTTP/HTTPS traffic.
- Multiple Apache (httpd) backend instances with replication, health checks, and isolated network.
- Docker internal network to simulate Kubernetes Services for inter-service communication.


### Architecture

```arduino
                 ┌─────────────────┐
                 │ Nginx Proxy     │
                 │ jwilder/nginx   │
                 │ VIRTUAL_HOST    │
                 └─────────┬───────┘
                           │
      ┌────────────────────┼─────────────────────┐
      │                    │                     │
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Module1 Server│    │ Module2 Server│    │ Module3 Server│
│ (2 replicas)  │    │ (2 replicas)  │    │ (2 replicas)  │
└───────────────┘    └───────────────┘    └───────────────┘

```

- Each module has 2 replicas, similar to replicated Pods in Kubernetes.
- The nginx-proxy automatically detects active containers based on the VIRTUAL_HOST environment variable, acting like a Kubernetes Ingress Controller.
- Health checks ensure traffic is only routed to healthy containers.


### 1️⃣ Dynamic Nginx Proxy

- Image: jwilder/nginx-proxy:latest
- Function: acts as a reverse proxy, automatically detecting containers via Docker Socket.
- Kubernetes equivalent: Ingress Controller distributing traffic to Services.

-  Volumes:
    -  SSL certificates (./certificates) for HTTPS.
    -  Docker Socket (/var/run/docker.sock) to dynamically detect containers and hosts.

### 2️⃣ Backend Modules (Module1, Module2, Module3)

- Image: httpd:latest (Apache)
- Volumes: website content (./example-moduleX)
- Replication: deploy.replicas: 2 → similar to a Deployment with multiple replicas in Kubernetes.
- Healthcheck: ensures only healthy containers receive traffic → equivalent to readiness/liveness probes in Kubernetes.
- VIRTUAL_HOST: defines the domain/subdomain for each module → nginx-proxy automatically generates the proxy configuration.

### Benefits of this Setup

1. Scalability: Increase replicas for more instances easily.
2. Resilience: Health checks prevent routing traffic to failing containers.
3. Dynamic Proxy: No need to manually edit Nginx configuration.
4. Kubernetes-like: Easily migratable to GCP GKE or any cluster, maintaining the same Deployment, Service, and Ingress logic.
5. Module Isolation: Each module is independent and can have its own domain/subdomain.


</details>

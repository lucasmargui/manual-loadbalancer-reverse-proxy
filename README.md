# Nginx Reverse Proxy Setup with Docker

Advanced Nginx reverse proxy architecture for multi-service Docker environments, optimized for maintainability, scalability, and local development.

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

```shell
               +------------------+
               |  nginx-proxy     |
               |  (Reverse Proxy) |  
               +--------+---------+
                        |
    ---------------------------------------------------
    |                     |                           |
+-------------+      +----------------+        +----------------+
| main-domain |      | module1        |        | module2        |
| server      |      | module1-server |        | module2-server |
+-------------+      +----------------+        +----------------+

```

- All services are Dockerized
- Nginx dynamically routes requests based on Host headers
- Backend containers serve isolated content for each domain/subdomai

--- 

## Docker Compose Architecture

`docker-compose.yml` defines isolated services with volume mounting for content:

```
version: "3.9"

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
      - main-domain-server
      - module1-main-domain-server
      - module2-main-domain-server

  main-domain-server:
    image: httpd:latest
    container_name: main-domain-server
    restart: always
    volumes:
      - ./main:/usr/local/apache2/htdocs/

  module1-main-domain-server:
    image: httpd:latest
    container_name: module1-main-domain-server
    restart: always
    volumes:
      - ./example-module1/module1:/usr/local/apache2/htdocs/

  module2-main-domain-server:
    image: httpd:latest
    container_name: module2-main-domain-server
    restart: always
    volumes:
      - ./example-module2/module2:/usr/local/apache2/htdocs/

```

- depends_on ensures backend services start before Nginx
- Apache serves static content via mounted volumes
- Volumes make local development iterative and fast

--- 

## Nginx Configuration

Reverse proxy configuration is modular: each server block corresponds to a domain/subdomain:

```
server {
    listen 80;
    server_name main-domain-example.online;

    location / {
        proxy_pass http://main-domain-server:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name module1.main-domain-example.online;

    location / {
        proxy_pass http://module1-main-domain-server:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name module2.main-domain-example.online;

    location / {
        proxy_pass http://module2-main-domain-server:80;

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
  
   <img width="1308" height="400" alt="image" src="https://github.com/user-attachments/assets/e2a85dae-2fb4-4956-92a6-ab387b18fefb" />


--- 
  
## Best Practices & Notes

- Use named Docker networks for isolation and explicit container communication
- Prefer environment variables for dynamic backend URLs
- Version control your Nginx configuration for rollback and audit
- Consider Docker healthchecks for backend containers to ensure Nginx only forwards to healthy services
- For production, combine with HTTPS, rate limiting, and caching strategies

--- 

This architecture is production-ready, highly maintainable, and extensible for microservices, modular applications, or multi-tenant hosting environments.

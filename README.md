# Nginx Reverse Proxy Setup with Docker 

A modern, modular, production-ready architecture for running **Nginx as a reverse proxy and load balancer** for multiple containerized services using Docker.

This improved documentation brings:
- Clearer architecture diagrams
- Detailed request flow
- Enhanced explanations for routing, proxying and load balancing
- Optimized folder structure and best practices
- Step-by-step local testing instructions

---

## 📌 Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Docker Compose Architecture](#docker-compose-architecture)
- [Nginx Configuration](#nginx-configuration)
- [Local Domain Simulation](#local-domain-simulation)
- [Request Flow](#request-flow)
- [Getting Started](#getting-started)
- [Best Practices](#best-practices)

---

## 🧭 Overview

The goal of this setup is to allow multiple independent modules (microservices, subdomains, or project sections) to run behind a **single Nginx reverse proxy**, while maintaining:

- Multi-domain/subdomain routing
- Clean project isolation
- Scalable load balancing
- Local development simulation of production behavior
- SSL readiness (Let’s Encrypt compatible)

### Components
- **nginx-proxy** → Reverse proxy router + load balancer
- **module1 / module2 / module3 servers** → Apache-based backend services
- **Local domains** → For easy routing during development

---

## 🏗 Architecture
```
                     +----------------------+
                     |     nginx-proxy      |
                     |  (Reverse Proxy LB)  |
                     +----------+-----------+
                                |
              -------------------------------------------
              |                    |                     |
      +---------------+   +----------------+   +----------------+
      | module1       |   | module2        |   | module3        |
      | Load Balanced |   | Load Balanced  |   | Load Balanced  |
      +---------------+   +----------------+   +----------------+
```

Each module contains **two backend servers**, providing fault tolerance and horizontal scalability.

Nginx receives all traffic and distributes incoming requests to the correct backend.

---

## 📦 Docker Compose Architecture

```yaml
version: "3.9"

networks:
  webnet:
    driver: bridge

services:
  ################################
  # Reverse Proxy (Nginx)
  ################################
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
      - module1-server-1
      - module1-server-2
      - module2-server-1
      - module2-server-2
      - module3-server-1
      - module3-server-2
    networks:
      - webnet

  ################################
  # Module 1 (Apache Servers)
  ################################
  module1-server-1:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module1:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module1-server-2:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module1:/usr/local/apache2/htdocs/
    networks:
      - webnet

  ################################
  # Module 2
  ################################
  module2-server-1:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module2:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module2-server-2:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module2:/usr/local/apache2/htdocs/
    networks:
      - webnet

  ################################
  # Module 3
  ################################
  module3-server-1:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module3:/usr/local/apache2/htdocs/
    networks:
      - webnet

  module3-server-2:
    image: httpd:latest
    restart: always
    volumes:
      - ./example-module3:/usr/local/apache2/htdocs/
    networks:
      - webnet
```

### Notes
- Each module has **two replicas** for load balancing.
- Code is mounted locally for real-time development.
- `depends_on` ensures backend readiness.

---

## ⚙️ Nginx Configuration

Reverse proxy rules for each subdomain/module, including load balancing groups.

```nginx
###########################
# Module 1
###########################
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

###########################
# Module 2
###########################
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

###########################
# Module 3
###########################
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

---

## 🖥 Local Domain Simulation

Edit your hosts file to simulate DNS:

```bash
127.0.0.1 main-domain-example.online
127.0.0.1 module1.main-domain-example.online
127.0.0.1 module2.main-domain-example.online
127.0.0.1 module3.main-domain-example.online
```

This enables local testing EXACTLY like a production environment.

---

## 🔄 Request Flow (Step-by-Step)

1. User accesses: `http://module1.main-domain-example.online`
2. Hosts file resolves to local machine (127.0.0.1)
3. Request hits **nginx-proxy**
4. Nginx reads `server_name` and selects the correct block
5. Nginx forwards the request to the correct upstream group:
   → `module1_backend`
6. Load balancer chooses one backend container (round-robin)
7. Apache inside the module container serves the file
8. Response returns:
   backend → nginx-proxy → browser

---

## 🚀 Getting Started

### 1. Clone the repository
```bash
git clone <repo-url>
cd <repo-directory>
```

### 2. Start the stack
```bash
docker compose up -d
```

### 3. Configure test domains
Update your OS hosts file with module domains.

### 4. Test in browser
- http://module1.main-domain-example.online
- http://module2.main-domain-example.online
- http://module3.main-domain-example.online

---

## ✅ Best Practices
- Use named Docker networks for clean isolation
- Mount volumes for fast development iteration
- Keep Nginx configs modular
- Add health checks to ensure Nginx avoids unhealthy containers
- Use Let's Encrypt for SSL termination
- Use caching for high performance

---


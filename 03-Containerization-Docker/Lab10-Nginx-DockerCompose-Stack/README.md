# 🌐 Lab 10: Nginx Reverse Proxy & Multi-Container Docker Compose Stack

## 🧠 Concept & Architecture ("What & Why")

### What is Nginx Reverse Proxy and Docker Compose?
- **Nginx Reverse Proxy**: An open-source, high-performance HTTP server that sits at the public boundary of your application network. It intercepts incoming client requests, serves static CSS/JS files directly from disk cache, and reverse-proxies dynamic web traffic to backend Gunicorn WSGI application servers.
- **Docker Compose**: A tool for defining and running multi-container Docker applications using a single declarative YAML file (`docker-compose.yml`). It automates bridge network creation, service dependency ordering (`depends_on`), environment variable injection, and persistent volume management across multiple containers.

```text
                                DOCKER COMPOSE MULTI-CONTAINER ARCHITECTURE
                                
  Client (Browser / Curl)
         |
         | HTTP Request (Port 80)
         v
  +-----------------------------------------------------------------------------------+
  |  DOCKER COMPOSE BRIDGE NETWORK (nti_network)                                      |
  |                                                                                   |
  |  +--------------------+    Reverse Proxy Pass     +--------------------+          |
  |  |  Nginx Container   | ========================> |  Django Container  |          |
  |  |  (nti_nginx_proxy) |    http://app:8000        |  (nti_django_app)  |          |
  |  |  Port 80:80        |                           |  Gunicorn WSGI     |          |
  |  +---------+----------+                           +---------+----------+          |
  |            | Serves Static Files directly                   |                     |
  |            v                                                |                     |
  |  +--------------------------------------------------+       |                     |
  |  |  Shared Named Volume (static_volume)             |       |                     |
  |  +--------------------------------------------------+       |                     |
  |                                                             |                     |
  |                                        +--------------------+------------------+  |
  |                                        | Encrypted Database | Session Cache    |  |
  |                                        v Connections        v Connections      |  |
  |                             +--------------------+     +--------------------+  |  |
  |                             | PostgreSQL 16 DB   |     | Redis 7 Cache      |  |  |
  |                             | (nti_postgres_db)  |     | (nti_redis_cache)  |  |  |
  |                             | Port 5432          |     | Port 6379          |  |  |
  |                             +--------------------+     +--------------------+  |  |
  +-----------------------------------------------------------------------------------+
```

---

## 🎯 Objectives

- Write Nginx reverse proxy configuration (`nginx/nginx.conf`) to proxy dynamic requests to Gunicorn and serve static assets directly.
- Define multi-container stack in `docker-compose.yml` linking Django, PostgreSQL 16, Redis 7, and Nginx.
- Configure persistent Docker named volumes (`postgres_data`, `static_volume`) to prevent data loss.
- Launch, test, and verify local stack execution (`docker compose up -d --build`).

---

## 📂 Project Structure

```text
03-Containerization-Docker/Lab10-Nginx-DockerCompose-Stack/
├── .env.example            # Environment variable template
├── docker-compose.yml      # Multi-container service composition specification
├── README.md               # Exhaustive step-by-step lab documentation
└── nginx/
    └── nginx.conf          # Nginx reverse proxy configuration blueprint
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Nginx Reverse Proxy Blueprint (`nginx/nginx.conf`)

```nginx
upstream django_app {
    server app:8000;
}

server {
    listen 80;
    server_name localhost;

    client_max_body_size 20M;

    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    location / {
        proxy_pass http://django_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}
```

#### Line-by-Line Breakdown:
- **`upstream django_app { server app:8000; }`**: Defines a backend server cluster named `django_app` pointing to service hostname `app` on Port 8000 (resolved via Docker Compose internal DNS).
- **`location /static/`**: Directly intercepts any URL request starting with `/static/` and serves files out of local container directory `/app/staticfiles/` with a 30-day browser cache header, avoiding expensive Django app processing.
- **`proxy_pass http://django_app`**: Forwards all dynamic HTTP requests to Gunicorn application server.
- **`proxy_set_header Host $host`**: Passes original HTTP host header to Django for correct domain URL routing.

---

### 2. Multi-Container Orchestration (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  app:
    build:
      context: ../Lab09-Django-MultiStage-Dockerfile
      dockerfile: Dockerfile
    container_name: nti_django_app
    environment:
      - DATABASE=postgres
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=nti_db
      - DB_USER=nti_admin
      - DB_PASSWORD=nti_password
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - static_volume:/app/staticfiles

  db:
    image: postgres:16-alpine
    container_name: nti_postgres_db
    environment:
      - POSTGRES_DB=nti_db
      - POSTGRES_USER=nti_admin
      - POSTGRES_PASSWORD=nti_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: nti_redis_cache
    ports:
      - "6379:6379"

  nginx:
    image: nginx:1.27-alpine
    container_name: nti_nginx_proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_volume:/app/staticfiles:ro
    depends_on:
      - app

volumes:
  postgres_data:
  static_volume:
```

---

## 📋 Step-by-Step Execution Guide

### Step 1: Launch Local Multi-Container Stack
```bash
cd 03-Containerization-Docker/Lab10-Nginx-DockerCompose-Stack
docker compose up -d --build
```

### Step 2: Inspect Running Stack Status
```bash
docker compose ps
```

### Step 3: Test Web Proxy Connectivity
```bash
curl -I http://localhost
```

---

## 🧪 Verification & Expected Terminal Output

```bash
docker compose ps
```

**Expected Output:**
```text
NAME               IMAGE                  COMMAND                  SERVICE   CREATED         STATUS         PORTS
nti_django_app     lab09-app              "/app/entrypoint.sh …"   app       1 minute ago    Up 1 minute    8000/tcp
nti_nginx_proxy    nginx:1.27-alpine      "/docker-entrypoint.…"   nginx     1 minute ago    Up 1 minute    0.0.0.0:80->80/tcp
nti_postgres_db    postgres:16-alpine     "docker-entrypoint.s…"   db        1 minute ago    Up 1 minute    0.0.0.0:5432->5432/tcp
nti_redis_cache    redis:7-alpine         "docker-entrypoint.s…"   redis     1 minute ago    Up 1 minute    0.0.0.0:6379->6379/tcp
```

---

## 🔗 Connection to Next Lab
With the local multi-container stack verified, **Lab 11** transitions application workloads into cloud-native Kubernetes manifests on AWS EKS!

# 🐳 Lab 09: Production-Grade Multi-Stage Dockerfile for Django & Gunicorn

## 🧠 Concept & Architecture ("What & Why")

### What is a Multi-Stage Docker Build?
A **Multi-Stage Docker Build** leverages multiple `FROM` instructions within a single `Dockerfile`. It separates the heavy build-time compilation environment from the minimal runtime execution environment.

1. **Stage 1 (`builder`)**: Installs compilers (`gcc`, `build-essential`), C-header libraries (`libpq-dev`), and compiles Python wheel packages into a temporary directory.
2. **Stage 2 (`runtime`)**: Starts from a clean, lightweight image (`python:3.13-slim`), copies ONLY the compiled Python binaries from Stage 1, drops root execution privileges, and runs the container as non-root `USER appuser`.

### Why use Multi-Stage builds & Non-Root Execution?
- **Extreme Size Reduction**: Reduces final image footprint from ~950MB down to ~160MB.
- **Vulnerability Surface Reduction**: Omitting compilers (`gcc`, `make`) prevents attackers from compiling malicious exploit code inside compromised containers.
- **Non-Root Security**: Running as `USER appuser` prevents container breakout exploits from obtaining root access to the host kernel.

```text
  +-------------------------------------+      +-------------------------------------+
  | STAGE 1: Builder (python:3.13-slim) |      | STAGE 2: Runtime (python:3.13-slim) |
  | - Installs gcc, make, libpq-dev     |      | - Copies compiled site-packages     |
  | - Compiles wheel dependencies       | ===> | - Non-root execution (USER appuser) |
  | - Image Footprint: ~950MB           |      | - Runs entrypoint.sh wait loop      |
  |                                     |      | - Image Footprint: ~160MB           |
  +-------------------------------------+      +-------------------------------------+
```

---

## 🎯 Objectives

- Write a production multi-stage `Dockerfile` (`python:3.13-slim`).
- Configure `entrypoint.sh` wait loop for PostgreSQL database readiness (`nc -z $DB_HOST $DB_PORT`).
- Automate Django migrations (`python manage.py migrate`) and static collection (`collectstatic`).
- Build, optimize, and test container execution locally.

---

## 📂 Project Structure

```text
03-Containerization-Docker/Lab09-Django-MultiStage-Dockerfile/
├── Dockerfile              # Production multi-stage Docker build specification
├── entrypoint.sh           # Container entrypoint script with DB wait loop
├── README.md               # Exhaustive step-by-step lab documentation
└── requirements.txt        # Python application dependencies
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Production Multi-Stage Dockerfile (`Dockerfile`)

```dockerfile
# STAGE 1: Builder
FROM python:3.13-slim as builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends     gcc build-essential libpq-dev     && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# STAGE 2: Runtime
FROM python:3.13-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends     libpq5 curl netcat-openbsd     && rm -rf /var/lib/apt/lists/*

COPY --from=builder /install /usr/local
COPY . .

RUN chmod +x /app/entrypoint.sh &&     useradd -m -s /bin/bash appuser &&     chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "3", "--timeout", "60", "core.wsgi:application"]
```

#### Line-by-Line Breakdown:
- **`FROM python:3.13-slim as builder`**: Declares Stage 1 compilation environment.
- **`COPY --from=builder /install /usr/local`**: Copies ONLY compiled Python binaries from Stage 1 into Stage 2.
- **`USER appuser`**: Switches container execution context from `root` to unprivileged user ID 1000 (`appuser`).
- **`exec "$@"` inside `entrypoint.sh`**: Replaces entrypoint process with Gunicorn WSGI server, ensuring proper Unix signal handling (`SIGTERM`).

---

## 📋 Step-by-Step Execution Guide

### Step 1: Build Docker Image
```bash
cd 03-Containerization-Docker/Lab09-Django-MultiStage-Dockerfile
docker build -t nti-django-app:1.0 .
```

### Step 2: Inspect Image Footprint & Layers
```bash
docker images nti-django-app:1.0
```

---

## 🧪 Verification & Expected Terminal Output

```bash
docker images nti-django-app:1.0
```

**Expected Output:**
```text
REPOSITORY       TAG       IMAGE ID       CREATED         SIZE
nti-django-app   1.0       a1b2c3d4e5f6   2 minutes ago   162MB
```

---

## 🔗 Connection to Next Lab
In **Lab 10**, we compose this multi-stage Django image into a complete multi-container stack with PostgreSQL, Redis, and Nginx using Docker Compose!

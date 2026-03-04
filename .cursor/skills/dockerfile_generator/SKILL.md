---
name: dockerfile-generator
version: 1.0.0
author: ZJLi2013
description: Generates optimized Dockerfiles for Python applications and frameworks (Flask, Django, FastAPI) with best practices (multi-stage builds, layer caching, security). Use when user asks to "create dockerfile", "dockerize app", "containerize", or "docker setup".
allowed-tools: [Read, Write, Glob, Grep, Shell]
link: https://mcpmarket.com/tools/skills/dockerfile-generator
---

# Dockerfile Generator (Python)

Generates production-ready Dockerfiles for Python projects with multi-stage builds, optimized layer caching, and security best practices.

## When to Use

- "Create a Dockerfile"
- "Dockerize my application"
- "Containerize this app"
- "Setup Docker"
- "Optimize Dockerfile"

## Instructions

### 1. Detect Project Type

Scan for Python project indicators:

```bash
# Python
[ -f "requirements.txt" ] && echo "requirements.txt found"
[ -f "Pipfile" ]          && echo "Pipfile found"
[ -f "pyproject.toml" ]   && echo "pyproject.toml found"
```

Read the dependency file to determine the framework (Flask, Django, FastAPI, etc.).

### 2. Generate Dockerfile by Framework

## FastAPI

```dockerfile
FROM python:3.11-slim AS base

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Dependencies
FROM base AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production
FROM base AS runner

RUN useradd -m -u 1001 appuser

COPY --from=deps /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=deps /usr/local/bin/uvicorn /usr/local/bin/uvicorn
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 3. Add .dockerignore

Create `.dockerignore` to reduce build context:

```dockerignore
# Version control
.git
.gitignore

# Python artifacts
__pycache__
*.pyc
*.pyo
*.pyd
.Python
venv
.venv
env
.env
*.egg-info
dist
build
.eggs

# IDE
.vscode
.idea
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
logs
*.log

# Test files
test
tests
.pytest_cache
.coverage
htmlcov

# Documentation
README.md
docs

# CI/CD
.github
.gitlab-ci.yml

# Environment
.env
.env.local
*.env

# Misc
.cache
tmp
temp
```

### 4. Optimization Techniques

**Layer caching:**
- Copy `requirements.txt` first, then install (cached layer)
- Copy source code last (changes frequently)

**Multi-stage builds:**
- `deps` stage: install all packages
- `runner` stage: copy only site-packages + source, run as non-root

**Minimize layers:**
- Combine `RUN` commands with `&&`
- Clean up `apt` cache in the same layer

**Use slim images:**
- `python:3.11-slim` is much smaller than `python:3.11`
- Alpine (`python:3.11-alpine`) is smallest but may need extra build deps

**Example optimization:**

```dockerfile
# ❌ BAD: Single stage, no caching, runs as root
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# ✅ GOOD: Multi-stage, cached deps, non-root user
FROM python:3.11-slim AS deps
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim AS runner
RUN useradd -m -u 1001 appuser
WORKDIR /app
COPY --from=deps /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --chown=appuser:appuser . .
USER appuser
CMD ["gunicorn", "app:app"]
```

### 5. Security Best Practices

**Don't run as root:**
```dockerfile
RUN useradd -m -u 1001 appuser
USER appuser
```

**Use specific versions:**
```dockerfile
# ❌ BAD
FROM python:latest

# ✅ GOOD
FROM python:3.11.8-slim
```

**Scan for vulnerabilities:**
```bash
docker scout cves myimage:latest
```

**Never copy secrets into the image — use environment variables at runtime:**
```bash
docker run -e SECRET_KEY=... myapp:latest
```

### 6. Docker Compose (Optional)

Offer to create `docker-compose.yml`:

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=myproject.settings.production
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

### 7. Provide Build Instructions

```bash
# Build image
docker build -t myapp:latest .

# Run container
docker run -p 8000:8000 myapp:latest

# With environment variables
docker run -p 8000:8000 \
  -e DATABASE_URL=postgresql://... \
  -e SECRET_KEY=... \
  myapp:latest

# Using docker-compose
docker-compose up -d

# View logs
docker-compose logs -f app

# Stop
docker-compose down
```

### Best Practices Checklist

- [ ] Multi-stage build (separate deps and runtime)
- [ ] Use `python:3.x-slim` or `python:3.x-alpine`
- [ ] Copy `requirements.txt` before source (layer caching)
- [ ] Don't run as root (create non-root user)
- [ ] Pin specific Python and package versions
- [ ] Include `.dockerignore`
- [ ] Set `PYTHONUNBUFFERED`, `PYTHONDONTWRITEBYTECODE`, `PIP_NO_CACHE_DIR`
- [ ] Combine `RUN` commands and clean up `apt` cache in same layer
- [ ] Use `COPY` instead of `ADD`
- [ ] Document `EXPOSE` ports
- [ ] Pass secrets via environment variables, never bake into image
- [ ] Security scan image

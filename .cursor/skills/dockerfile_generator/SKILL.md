---
name: dockerfile-generator
version: 1.0.0
author: ZJLi2013
description: Generates optimized Dockerfiles for Python applications and frameworks (Flask, Django, FastAPI) with best practices (multi-stage builds, layer caching, security). Use when user asks to "create dockerfile", "dockerize app", "containerize", or "docker setup".
allowed-tools: [Read, Write, Glob, Grep, Shell]
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

Scan for Python project indicators and read the dependency file to determine the framework.

### 2. Generate Dockerfile by Framework

Use multi-stage builds, slim base images, non-root users, and proper layer caching.

### 3. Add .dockerignore

Create `.dockerignore` to reduce build context.

### 4. Provide Build Instructions

```bash
docker build -t myapp:latest .
docker run -p 8000:8000 myapp:latest
```

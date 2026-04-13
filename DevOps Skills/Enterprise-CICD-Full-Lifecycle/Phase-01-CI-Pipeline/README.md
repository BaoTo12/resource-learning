# 🛠️ PHASE 1: CONTINUOUS INTEGRATION (CI)
### Setting Up the Engine | GitHub Actions + Docker

---

## 🎯 Objective
Automate the build, test, and containerization of the **Reservation Platform**. Every push to `main` must produce a high-quality, scanned, and ready-to-deploy Docker image.

---

## 📂 1. Containerizing the App (The Dockerfile)

Your app needs to be "container-native." If it's a Node.js app, create a `Dockerfile` in the root of your project:

```dockerfile
# Use a lightweight Node image
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package info and install dependencies
COPY package*.json ./
RUN npm install --production

# Copy source code
COPY . .

# Multi-stage build for a smaller, more secure final image
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app /app

# Expose port and start
EXPOSE 3000
CMD ["node", "index.js"]
```

> **Pro Tip:** Use `.dockerignore` to exclude `node_modules`, `.git`, and sensitive `.env` files.

---

## 🎡 2. The CI Workflow (GitHub Actions)

Create a directory: `.github/workflows/` and add a file: `ci.yml`.

### The "Full Lifecycle" CI Pipeline
This pipeline does more than just build; it ensures quality and security.

```yaml
name: CI Pipeline - Reservation Platform

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install Dependencies
        run: npm install

      - name: Run Unit Tests
        run: npm test

      - name: Lint Code
        run: npm run lint

  docker-build:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' # Only push image on main branch
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to GitHub Container Registry (GHCR)
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/reservation-app:latest
            ghcr.io/${{ github.repository }}/reservation-app:${{ github.sha }}
```

---

## 🛡️ 3. Security Scanning (Free & Essential)

Don't push "dirty" images. Use **Trivy** to scan for vulnerabilities within your CI pipeline.

**Add this step before pushing:**
```yaml
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'ghcr.io/${{ github.repository }}/reservation-app:${{ github.sha }}'
          format: 'table'
          exit-code: '1' # Fail the build if vulnerabilities are found!
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'
```

---

## ✅ Checklist for Phase 1
1. [ ] `Dockerfile` added to the repo.
2. [ ] Unit tests pass locally.
3. [ ] GitHub Action triggered on push.
4. [ ] Image successfully pushed to **GHCR** (free!) or **Docker Hub**.

---

### [Next: Phase 2 — Packaging with Helm →](../Phase-02-Helm-Chart/README.md)

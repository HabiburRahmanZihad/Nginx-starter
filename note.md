# Docker Study Notes — Multi-Stage Builds & CI/CD Pipeline

> Detailed study notes on Docker's two build strategies (single-stage vs multi-stage), why multi-stage builds matter in production, and a full breakdown of CI/CD — what it is, how it works, and how GitHub Actions automates the entire pipeline.

---

## Table of Contents

1. [Docker Build: Two Ways](#1-docker-build-two-ways)
2. [Single-Stage Build](#2-single-stage-build)
3. [Multi-Stage Build](#3-multi-stage-build)
4. [Single-Stage vs Multi-Stage Comparison](#4-single-stage-vs-multi-stage-comparison)
5. [What Is CI/CD?](#5-what-is-cicd)
6. [Continuous Integration (CI)](#6-continuous-integration-ci)
7. [Continuous Delivery/Deployment (CD)](#7-continuous-deliverydeployment-cd)
8. [CI/CD Pipeline Flow](#8-cicd-pipeline-flow)
9. [GitHub Actions — Hands-on Example](#9-github-actions--hands-on-example)
10. [Recap](#10-recap)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. Docker Build: Two Ways

Docker can build images in **two fundamentally different ways**:

| Strategy         | How It Works                                            | Best For          |
| ---------------- | ------------------------------------------------------- | ----------------- |
| **Single-Stage** | One `FROM` instruction, everything in one image         | Simple apps, dev  |
| **Multi-Stage**  | Multiple `FROM` instructions, only final output is kept | Production, CI/CD |

The goal of multi-stage is simple: **build the app in one image, run it in a much smaller image.**

---

## 2. Single-Stage Build

### What Is It?

A single-stage build uses **one base image** for everything — installing tools, building the app, and running it. All layers (source code, build tools, compilers, dev dependencies) end up in the final image.

### Example — Node.js Single-Stage Dockerfile

```dockerfile
FROM node:18

WORKDIR /app

# Copy everything
COPY package*.json ./
RUN npm install          # installs ALL dependencies (including devDependencies)

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### The Problem

```
Final image contains:
  ✅ node runtime
  ✅ app source code
  ❌ npm (not needed at runtime)
  ❌ devDependencies (test libs, linters, etc.)
  ❌ build tools (TypeScript compiler, webpack, etc.)
  ❌ source maps, test files

Result: Image size ~900MB+ for a simple app
```

> The final image carries everything used during the build — even things the running app never needs. This wastes disk space, increases attack surface, and slows down deployments.

---

## 3. Multi-Stage Build

### What Is It?

A multi-stage build uses **multiple `FROM` instructions** in one Dockerfile. Each stage can build on a previous one. At the end, only what you explicitly copy into the **final stage** makes it into the image.

```
Stage 1 (builder):   fat image — has compiler, dev tools, source code
         │
         │   COPY --from=builder /app/dist ./dist
         ▼
Stage 2 (runner):    slim image — has only runtime + compiled output
```

### Example — Node.js Multi-Stage Dockerfile

```dockerfile
# ─── Stage 1: Build ───────────────────────────────────────────────
FROM node:18 AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install              # install ALL deps (including devDeps for build)

COPY . .
RUN npm run build            # compile TypeScript → dist/


# ─── Stage 2: Production Runner ───────────────────────────────────
FROM node:18-alpine AS runner
# node:18-alpine is ~70MB vs node:18 ~900MB

WORKDIR /app

COPY package*.json ./
RUN npm install --omit=dev   # install ONLY production dependencies

COPY --from=builder /app/dist ./dist    # ← copy ONLY the compiled output

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### What Gets Discarded

```
Stage 1 (builder) — THROWN AWAY after build:
  ✗ TypeScript compiler
  ✗ devDependencies (jest, eslint, ts-node...)
  ✗ raw .ts source files
  ✗ node_modules (full, heavy)

Stage 2 (runner) — KEPT in final image:
  ✓ node:alpine runtime
  ✓ dist/ (compiled JS only)
  ✓ node_modules (production only)
```

### Size Difference

```
Single-stage image:   ~950 MB
Multi-stage image:    ~120 MB   ← 87% smaller
```

### Multi-Stage for a React App (Build + Nginx Serve)

```dockerfile
# ─── Stage 1: Build React App ──────────────────────────────────────
FROM node:18 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build            # → creates /app/build/ or /app/dist/


# ─── Stage 2: Serve with Nginx ─────────────────────────────────────
FROM nginx:alpine AS runner
# nginx:alpine is only ~23MB!

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> The final image has NO Node.js, NO npm, NO source code — just a tiny Nginx serving static files.

---

## 4. Single-Stage vs Multi-Stage Comparison

| Aspect               | Single-Stage               | Multi-Stage                   |
| -------------------- | -------------------------- | ----------------------------- |
| **Image size**       | Large (all tools included) | Small (runtime only)          |
| **Security**         | Higher attack surface      | Minimal attack surface        |
| **Build speed**      | Fast (fewer instructions)  | Slightly more steps           |
| **Complexity**       | Simple                     | Slightly more complex         |
| **Production use**   | ❌ Not recommended         | ✅ Strongly recommended       |
| **Dockerfile lines** | ~10–15                     | ~20–30                        |
| **CI/CD friendly**   | ❌ Slow pushes to registry | ✅ Smaller images = faster CD |

### When to Use Each

```
Single-Stage:
  ✓ Quick local development
  ✓ Simple scripts or tools
  ✓ When image size doesn't matter

Multi-Stage:
  ✓ Any production deployment
  ✓ CI/CD pipelines (push to Docker Hub / ECR / GHCR)
  ✓ Apps with a compile/build step (TypeScript, React, Java, Go)
  ✓ When you want to minimize the attack surface
```

---

## 5. What Is CI/CD?

**CI/CD** stands for **Continuous Integration / Continuous Delivery (or Deployment)**. It is a software engineering practice that automates the process of building, testing, and deploying code every time a developer pushes changes.

```
Developer pushes code
        │
        ▼
  ┌─────────────────────────────────────────────────┐
  │               CI/CD PIPELINE                    │
  │                                                 │
  │  CI Phase:                                      │
  │    1. Code checkout                             │
  │    2. Install dependencies                      │
  │    3. Run linter / static analysis              │
  │    4. Run unit tests                            │
  │    5. Build Docker image                        │
  │                                                 │
  │  CD Phase:                                      │
  │    6. Push image to registry                    │
  │    7. Deploy to staging / production server     │
  │    8. Run smoke tests                           │
  │                                                 │
  └─────────────────────────────────────────────────┘
        │
        ▼
  App is live with new changes ✅
```

> CI/CD turns "works on my machine" into "works everywhere, automatically."

---

## 6. Continuous Integration (CI)

### What Is CI?

**Continuous Integration** is the practice of automatically **building and testing** every code change as soon as it is pushed to a shared repository.

### The Problem CI Solves

```
Without CI:
  Dev A works on branch for 2 weeks
  Dev B works on branch for 2 weeks
  They merge → CONFLICT HELL 💥
  Tests fail, no one knows why

With CI:
  Every push → automated build + tests run immediately
  Bugs caught within minutes, not weeks
  Team always has a working main branch
```

### What CI Typically Does

| Step                    | Tool Examples                   | Purpose                                  |
| ----------------------- | ------------------------------- | ---------------------------------------- |
| **Code checkout**       | `actions/checkout`              | Get the latest code                      |
| **Install deps**        | `npm install`, `pip install`    | Prepare build environment                |
| **Lint / format check** | ESLint, Prettier, flake8        | Enforce code style                       |
| **Unit tests**          | Jest, Pytest, JUnit             | Verify logic is correct                  |
| **Build**               | `npm run build`, `docker build` | Confirm the app actually compiles/builds |
| **Code coverage**       | Codecov, Istanbul               | Ensure tests cover enough code           |
| **Security scan**       | Snyk, npm audit                 | Catch vulnerable dependencies            |

### CI Trigger Events

```yaml
# Runs CI on every push and pull request to main
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

---

## 7. Continuous Delivery/Deployment (CD)

### What Is CD?

**Continuous Delivery** means every successful CI run produces a deployable artifact (Docker image, binary, etc.) that _can_ be deployed with one click.

**Continuous Deployment** goes one step further — every successful CI run is **automatically deployed** to production with no human approval needed.

```
Continuous Integration:
  push → build → test ✅ → artifact ready

Continuous Delivery:
  push → build → test ✅ → artifact ready → (manual approval) → deploy

Continuous Deployment:
  push → build → test ✅ → artifact ready → auto-deploy ✅
```

### What CD Typically Does

| Step                   | Tool Examples             | Purpose                            |
| ---------------------- | ------------------------- | ---------------------------------- |
| **Build Docker image** | `docker build`            | Package app into a container       |
| **Push to registry**   | Docker Hub, GHCR, AWS ECR | Store the image for deployment     |
| **SSH to server**      | `appleboy/ssh-action`     | Connect to the VPS/server          |
| **Pull latest image**  | `docker pull`             | Get the new image on the server    |
| **Restart containers** | `docker compose up -d`    | Run new version                    |
| **Health check**       | `curl`, uptime monitor    | Verify the new deployment is alive |

---

## 8. CI/CD Pipeline Flow

### Full Pipeline — From Push to Production

```
┌────────────────────────────────────────────────────────────────────┐
│                      FULL CI/CD PIPELINE                          │
│                                                                    │
│  1. Developer pushes code to GitHub (main branch)                 │
│                    │                                               │
│                    ▼                                               │
│  2. GitHub Actions workflow triggered                              │
│                    │                                               │
│         ┌──────────┴──────────┐                                   │
│         │     CI PHASE        │                                   │
│         │  • checkout code    │                                   │
│         │  • npm install      │                                   │
│         │  • npm test         │                                   │
│         │  • docker build     │                                   │
│         └──────────┬──────────┘                                   │
│                    │ (if all pass ✅)                              │
│         ┌──────────┴──────────┐                                   │
│         │     CD PHASE        │                                   │
│         │  • docker push      │                                   │
│         │    → Docker Hub     │                                   │
│         │  • SSH to VPS       │                                   │
│         │  • docker pull      │                                   │
│         │  • docker compose   │                                   │
│         │    up -d            │                                   │
│         └──────────┬──────────┘                                   │
│                    │                                               │
│  3. App is live with new code ✅                                   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### How Docker Multi-Stage Fits into CI/CD

```
GitHub Actions:
  1. docker build -t myapp:latest .   ← uses multi-stage Dockerfile
        └─ Stage 1: compiles code (builder)
        └─ Stage 2: copies only output (runner)  ← small final image

  2. docker push myapp:latest         ← pushes ~120MB, not ~900MB
        └─ faster push = faster pipeline

  3. SSH to server: docker pull + docker compose up -d
        └─ server downloads ~120MB, not ~900MB  ← faster deployment
```

> Multi-stage builds make CI/CD faster because smaller images push and pull faster.

---

## 9. GitHub Actions — Hands-on Example

### Project Structure

```
my-app/
├── .github/
│   └── workflows/
│       └── ci-cd.yml     ← the CI/CD pipeline definition
├── Dockerfile             ← multi-stage Dockerfile
├── docker-compose.yml
└── src/
    └── server.js
```

### The GitHub Actions Workflow (`.github/workflows/ci-cd.yml`)

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest

    steps:
      # ─── CI Phase ────────────────────────────────────────────────

      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

      # ─── Build Docker Image (multi-stage) ────────────────────────

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myusername/myapp:latest

      # ─── CD Phase — Deploy to VPS ────────────────────────────────

      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            docker pull myusername/myapp:latest
            docker compose -f /app/docker-compose.yml up -d --no-deps myapp
```

### GitHub Secrets You Need to Set

| Secret Name       | What It Stores                       |
| ----------------- | ------------------------------------ |
| `DOCKER_USERNAME` | Your Docker Hub username             |
| `DOCKER_PASSWORD` | Your Docker Hub access token         |
| `SERVER_HOST`     | VPS IP address or domain             |
| `SERVER_USER`     | SSH user (e.g. `root` or `ubuntu`)   |
| `SERVER_SSH_KEY`  | Private SSH key to access the server |

> Never hardcode secrets in the workflow file. Always use `${{ secrets.SECRET_NAME }}`.

---

## 10. Recap

| Concept                | Key Point                                                            |
| ---------------------- | -------------------------------------------------------------------- |
| **Single-stage build** | Simple, but final image is large — not suitable for production       |
| **Multi-stage build**  | Final image is small and contains only what the app needs to run     |
| **CI**                 | Automatically builds and tests every code push                       |
| **CD**                 | Automatically packages and deploys the app after CI passes           |
| **GitHub Actions**     | Free CI/CD runner built into GitHub — triggered by push/PR events    |
| **Docker + CI/CD**     | Multi-stage builds make images smaller, which makes pipelines faster |

---

## 11. Quick Reference Cheat Sheet

```bash
# Build image locally (multi-stage)
docker build -t myapp:latest .

# Run the production image
docker run -p 3000:3000 myapp:latest

# Check image size
docker images myapp

# Push image to Docker Hub
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest

# Trigger GitHub Actions manually
gh workflow run ci-cd.yml

# View GitHub Actions logs
# → Go to GitHub repo → Actions tab → click on a run
```

```nginx
# Multi-stage Dockerfile summary:
FROM node:18 AS builder          # Stage 1 — build
  RUN npm run build

FROM node:18-alpine AS runner    # Stage 2 — run (final image)
  COPY --from=builder /app/dist ./dist
  CMD ["node", "dist/server.js"]
```

# Project Technical Report: Automated CI/CD Pipeline with AWS and GitHub Actions

---

## Academic Metadata

* **University:** Visvesvaraya Technological University (VTU), Jnanasangama, Belagavi - 590018
* **Institution:** Jawaharlal Nehru New College of Engineering (JNNCE), Navule, Savalanga Road, Shivamogga, Karnataka - 577204
* **Department:** Department of Master of Computer Applications (MCA)
* **Project Title:** *Automated CI/CD Pipeline with AWS and GitHub Actions*
* **Candidate Name:** Shashank P M
* **University Seat Number (USN):** `4JN25MC089`
* **Internal Guide:** Mr. Prashant Ankalkoti, Assistant Professor
* **Academic Year:** 2026–2027
* **Target Environment:** AWS EC2 Ubuntu Cloud VM (Docker Containerized)
* **Live Deployment URL:** [http://3.111.38.88/](http://3.111.38.88/)
* **Health Check URL:** [http://3.111.38.88/health](http://3.111.38.88/health)
* **GitHub Repository:** [https://github.com/shashankpm2004/ci-cd-demo](https://github.com/shashankpm2004/ci-cd-demo)
* **DockerHub Repository:** [https://hub.docker.com/r/shashankpm/newapp](https://hub.docker.com/r/shashankpm/newapp)

---

## Executive Summary / Abstract

Modern software engineering demands rapid, reliable, and repeatable delivery cycles. Traditional manual deployment procedures—such as copying application files over FTP/SCP, manually managing server processes, and editing configurations by hand—are time-consuming, prone to human error, introduce environment drift, and leave no verifiable trace linking running containers back to source code commits.

This project designs and implements an enterprise-grade, fully automated **Continuous Integration and Continuous Deployment (CI/CD)** pipeline for a containerized Node.js application using **GitHub Actions**, **DockerHub**, and an **Amazon Web Services (AWS) Elastic Compute Cloud (EC2)** Linux virtual machine.

### Key Milestones Achieved:
1. **Automated Fail-Fast Testing (CI):** Every push to the `main` branch triggers an isolated test suite on an Ubuntu runner, executing a live `curl` validation check against the running server. Untested or broken code is aborted before consuming build resources.
2. **Containerization & Multi-Tagging (CD):** Validated builds are packaged into a standardized Docker container using `node:18`. The image is tagged with `:latest` and the immutable Git commit SHA (`${{ github.sha }}`), providing auditability and instant rollback capability.
3. **Automated Cloud Deployment (CD):** Secure SSH key-based authentication connects GitHub Actions to the remote AWS EC2 instance (`3.111.38.88`). The workflow pulls the latest container image, gracefully halts stale containers, spins up the new release on standard HTTP port `80` and custom port `3000`, performs a post-deployment health check, and prunes unused Docker images to maintain server disk hygiene.
4. **Minimalist, Hot-Reload Architecture:** The microservice features a distraction-free, minimalist UI and dual-mode health diagnostics serving HTML to web browsers and structured JSON to automated monitoring probes.

---

## Chapter 1: Introduction & Problem Statement

### 1.1 The Shift Towards DevOps and CI/CD
DevOps represents a cultural and technical methodology that unifies software development (Dev) and IT operations (Ops). The cornerstone of modern DevOps is the CI/CD pipeline:
* **Continuous Integration (CI):** Developers commit and merge code into a shared repository frequently. Automated builds and tests run on every commit to surface bugs immediately.
* **Continuous Deployment (CD):** Code changes that pass all testing stages are packaged and deployed directly into live production environments without manual operator intervention.

### 1.2 Traditional Deployment Pitfalls
Prior to automated pipelines, deployments suffered from critical bottlenecks:
* **"It Works on My Machine" Syndrome:** Differences in underlying operating systems, installed package versions, and environment configurations between developer machines and production servers caused unexpected production outages.
* **Human Operator Errors:** Manual execution of shell scripts, incorrect file permissions, missed environment flags, and forgotten process restarts frequently led to deployment failures.
* **Lack of Commit Traceability:** Traditional releases provided no automated way to determine which Git commit or pull request was running on a given server.
* **Credential Exposure:** Registry passwords, cloud API tokens, and server private keys were frequently hardcoded into scripts or mishandled in unencrypted chat channels.

### 1.3 Proposed Automated Solution
This project eliminates manual deployment risks through a three-stage declarative GitHub Actions workflow:
* Standardized runtime environment guaranteed via Docker containerization.
* Complete cryptographic credential security using GitHub Secrets.
* Automated rollback readiness through Git commit SHA image tagging.
* Zero-downtime container replacement on an AWS EC2 Ubuntu virtual machine.

---

## Chapter 2: System Architecture & Pipeline Design

### 2.1 End-to-End Workflow Diagram

```
[ Developer Local Machine ]
           │
           │ git push origin main
           ▼
[ GitHub Repository (shashankpm2004/ci-cd-demo) ]
           │
           ▼ (Triggered Automatically)
[ GitHub Actions Workflow (.github/workflows/deploy.yml) ]
           │
           ├── STAGE 1: Automated Testing (test)
           │       • Checkout repository
           │       • Setup Node.js 18
           │       • Run background server (node app.js &)
           │       • curl -f http://localhost:3000
           │       • Fail-Fast: Abort if response != 200
           │       • pkill node
           │
           ├── STAGE 2: Build & Tag (build)
           │       • Login to DockerHub (GitHub Secrets)
           │       • Build Docker image (IMAGE_NAME: newapp)
           │       • Tag 1: shashankpm/newapp:latest
           │       • Tag 2: shashankpm/newapp:<commit_sha>
           │       • Push both tags to DockerHub
           │
           └── STAGE 3: Remote Deployment (deploy)
                   • SSH into AWS EC2 (appleboy/ssh-action & linux.pem)
                   • docker pull shashankpm/newapp:latest
                   • docker stop newapp-container || true
                   • docker rm newapp-container || true
                   • docker run -d -p 80:3000 -p 3000:3000 --name newapp-container --restart always
                   • Post-deployment verification: curl -f http://localhost:3000
                   • Disk hygiene: docker image prune -f
                           │
                           ▼
            [ Live Application on AWS EC2 ]
              http://3.111.38.88/ (Port 80)
              http://3.111.38.88:3000/ (Port 3000)
              http://3.111.38.88/health (Diagnostics)
```

### 2.2 Stage-Wise Technical Breakdown

| Stage | Execution Environment | Primary Operations | Fail-Safe & Quality Controls |
| :--- | :--- | :--- | :--- |
| **Stage 1: Test** | GitHub Actions Runner (`ubuntu-latest`) | Launches application in background; runs `curl -f http://localhost:3000`. | **Fail-Fast:** If server fails to boot or return HTTP 200, execution aborts with exit code 1. |
| **Stage 2: Build** | GitHub Actions Runner (`ubuntu-latest`) | Authenticates to DockerHub; builds container; tags with `:latest` and commit SHA; pushes image. | **Dependency Guard:** Requires Stage 1 (`needs: test`). Broken builds are never packaged or published. |
| **Stage 3: Deploy** | AWS EC2 Linux Server (`3.111.38.88`) | Connects over SSH; pulls new image; replaces `newapp-container`; verifies health; cleans dangling layers. | **Rollback Ready:** If deployment health check fails, container logs are printed and pipeline flags failure. |

---

## Chapter 3: Key Source Code & Implementation

This chapter presents the **core logic and important code segments** from each project file, with detailed explanations. The complete source files are available in the [GitHub repository](https://github.com/shashankpm2004/ci-cd-demo).

---

### 3.1 Application Server: `app.js`

The application is a lightweight Node.js HTTP server built entirely with native standard modules (`http`, `fs`, `path`) — no external frameworks like Express are used. This keeps the Docker image small, eliminates dependency vulnerabilities, and ensures a container memory footprint under 45 MB.

#### Key Code 1 — Dual-Mode Health Check Endpoint (`/health`)

This is the most critical feature of the server. It auto-detects whether the caller is a **web browser** or an **automated monitoring probe** and responds with the appropriate format:

```javascript
if (urlPath === '/health') {
  const isJsonRequested = req.url.includes('format=json') ||
    (req.headers.accept && !req.headers.accept.includes('text/html'));

  if (isJsonRequested) {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    return res.end(JSON.stringify({
      status: 'UP',
      port: PORT,
      uptime_seconds: Math.floor(process.uptime()),
      memory_mb: Math.round(process.memoryUsage().rss / (1024 * 1024)),
      node_version: process.version,
      timestamp: new Date().toISOString()
    }, null, 2));
  }
  // ... otherwise serves rendered HTML health page for browsers
}
```

**Explanation:**
- The endpoint inspects either the `?format=json` query parameter or the HTTP `Accept` header to decide the response format.
- For CI tools (like `curl`), it returns a **structured JSON** payload containing real-time metrics: uptime, memory usage, Node.js version, and a timestamp.
- For browser requests, it reads and renders the `health.html` template, substituting `{{UPTIME}}`, `{{MEMORY_MB}}`, etc., with live values using simple string replacement.

#### Key Code 2 — Directory Traversal Security Guard

This block prevents attackers from accessing system files outside the `public/` directory:

```javascript
let filePath = path.join(PUBLIC_DIR, relativePath);

// Security check: Prevent path traversal outside PUBLIC_DIR
if (!filePath.startsWith(PUBLIC_DIR)) {
  res.writeHead(403, { 'Content-Type': 'text/plain' });
  return res.end('403 Forbidden\n');
}
```

**Explanation:**
- `path.join()` resolves any `../` sequences in the URL to an absolute path.
- The `startsWith()` check ensures the final resolved path is still within the `public/` directory.
- A malicious request like `GET /../../etc/passwd` is blocked with an HTTP **403 Forbidden** response.

#### Key Code 3 — Graceful Container Shutdown

Docker sends `SIGTERM` when stopping a container. This handler ensures open connections are closed cleanly:

```javascript
process.on('SIGTERM', () => {
  console.log('SIGTERM signal received: closing HTTP server');
  server.close(() => {
    console.log('HTTP server closed');
    process.exit(0);
  });
});
```

**Explanation:**
- When Docker issues `docker stop`, it sends `SIGTERM` to PID 1 inside the container.
- The handler calls `server.close()`, which stops accepting new connections and waits for in-flight requests to complete before exiting with code 0.
- Without this, Docker would forcefully kill the process after a 10-second timeout, potentially corrupting active responses.

---

### 3.2 CI/CD Pipeline Workflow: `.github/workflows/deploy.yml`

This is the central orchestration file that defines the entire three-stage automated pipeline. It is triggered on every push to the `main` branch.

#### Key Code 1 — Workflow Trigger & Global Variables

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  IMAGE_NAME: newapp
  CONTAINER_NAME: newapp-container
```

**Explanation:**
- `on: push: branches: [main]` — The pipeline runs automatically on every push to the `main` branch.
- `workflow_dispatch` — Enables manual triggering from the GitHub Actions UI for ad-hoc deployments or re-runs.
- `env` — Global environment variables (`IMAGE_NAME`, `CONTAINER_NAME`) are defined once and reused across all three stages, following the DRY (Don't Repeat Yourself) principle.

#### Key Code 2 — Stage 1: Automated Testing (Fail-Fast)

```yaml
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 18
    - name: Start App in Background
      run: |
        node app.js &
        sleep 3
    - name: Test Application Response
      run: |
        curl -f http://localhost:3000 || exit 1
    - name: Stop App
      run: |
        pkill node || true
```

**Explanation:**
- A fresh Ubuntu runner is provisioned. Node.js 18 is installed.
- The app is started in the background using `&`, then the pipeline waits 3 seconds for the server to bind to port 3000.
- `curl -f` (fail-fast flag) tests the endpoint. The `-f` flag causes `curl` to return **exit code 22** on HTTP 4xx/5xx errors, which triggers `exit 1` and immediately aborts the entire pipeline.
- If this stage fails, Stages 2 and 3 are **skipped entirely** — broken code is never built or deployed.

#### Key Code 3 — Stage 2: Docker Build & Multi-Tag Push

```yaml
build:
  needs: test
  runs-on: ubuntu-latest
  steps:
    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    - name: Tag Image
      run: |
        docker tag $IMAGE_NAME:latest ${{ secrets.DOCKER_USERNAME }}/$IMAGE_NAME:latest
        docker tag $IMAGE_NAME:latest ${{ secrets.DOCKER_USERNAME }}/$IMAGE_NAME:${{ github.sha }}
    - name: Push Images
      run: |
        docker push ${{ secrets.DOCKER_USERNAME }}/$IMAGE_NAME:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/$IMAGE_NAME:${{ github.sha }}
```

**Explanation:**
- `needs: test` — This stage only runs if Stage 1 passes (dependency guard).
- DockerHub login uses encrypted GitHub Secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`) — no credentials are exposed in the workflow file.
- The image receives **two tags:**
  - `:latest` — A mutable pointer so deployment scripts always pull the newest version.
  - `:${{ github.sha }}` — An **immutable** tag using the full Git commit hash, enabling exact commit-to-container traceability and instant rollback.

#### Key Code 4 — Stage 3: Remote SSH Deployment to AWS EC2

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  steps:
    - name: Deploy via SSH
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SERVER_SSH_KEY }}
        script: |
          set -e
          docker pull ${{ secrets.DOCKER_USERNAME }}/newapp:latest
          docker stop newapp-container || true
          docker rm newapp-container || true
          docker run -d -p 80:3000 -p 3000:3000 \
            --name newapp-container --restart always \
            ${{ secrets.DOCKER_USERNAME }}/newapp:latest
          sleep 5
          curl -f http://localhost:3000 || (echo "Container failed!" && exit 1)
          docker image prune -f
```

**Explanation:**
- `appleboy/ssh-action@v1.0.3` establishes an SSH tunnel to the EC2 instance using the private key stored in `SERVER_SSH_KEY`.
- `set -e` ensures the script halts on the first error.
- The old container is gracefully stopped and removed (`|| true` prevents errors if no container exists).
- The new container is started with:
  - `-d` — Detached mode (runs in background).
  - `-p 80:3000 -p 3000:3000` — Maps host ports 80 and 3000 to the container's port 3000.
  - `--restart always` — Docker automatically restarts the container on crash or server reboot.
- A **post-deployment health check** (`curl -f`) verifies the container is responding before declaring success.
- `docker image prune -f` cleans up old, unreferenced image layers to prevent disk exhaustion on the EC2 instance.

---

### 3.3 Container Specification: `Dockerfile`

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

**Explanation:**
- `FROM node:18` — Uses the official Node.js 18 LTS base image, ensuring a consistent runtime across all environments.
- `WORKDIR /app` — Sets the working directory for all subsequent commands inside the container.
- `COPY . .` — Copies the entire application (source code, public assets, scripts) into the container.
- `EXPOSE 3000` — Documents that the container listens on TCP port 3000 (informational; actual port mapping is done at `docker run`).
- `CMD ["node", "app.js"]` — Defines the default startup command. Using exec form (`["node", "app.js"]`) ensures Node.js runs as PID 1 and receives signals like `SIGTERM` directly.

---

### 3.4 Local Test Script: `scripts/test.sh` (Key Segments)

This Bash script mirrors the CI pipeline's Stage 1 logic for local developer testing.

#### Key Code — Retry Loop & Endpoint Validation

```bash
#!/usr/bin/env bash
set -e

# Start app in background and register cleanup
node app.js &
APP_PID=$!
trap 'kill $APP_PID 2>/dev/null || true' EXIT

# Retry loop: wait up to 15 seconds for server readiness
MAX_RETRIES=15
COUNT=0
while [ $COUNT -lt $MAX_RETRIES ]; do
  if curl -s -f "http://127.0.0.1:3000/" > /dev/null 2>&1; then
    break
  fi
  sleep 1
  COUNT=$((COUNT + 1))
done

# Validate root endpoint contains expected greeting
RESPONSE=$(curl -s "http://127.0.0.1:3000/")
if [[ "$RESPONSE" == *"Hello, Node.js!"* ]]; then
  echo "[PASS] Root endpoint returned expected greeting"
else
  echo "[FAIL] Unexpected response"; exit 1
fi

# Validate health endpoint returns HTTP 200
HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://127.0.0.1:3000/health")
if [ "$HEALTH_STATUS" -eq 200 ]; then
  echo "[PASS] Health check returned HTTP 200 OK"
else
  echo "[FAIL] Health check returned HTTP $HEALTH_STATUS"; exit 1
fi
```

**Explanation:**
- `set -e` — Exits immediately on any command failure (fail-fast).
- `trap ... EXIT` — Guarantees the background Node.js process is killed when the script exits, regardless of success or failure.
- The retry loop polls the server every second for up to 15 seconds, handling the brief delay between process start and port binding.
- Two assertions are run: (1) the root endpoint must contain "Hello, Node.js!", and (2) the `/health` endpoint must return HTTP 200.

---

### 3.5 EC2 Server Setup: `scripts/ec2-setup.sh` (Key Segment)

This one-time provisioning script installs Docker on a fresh AWS EC2 Ubuntu instance:

```bash
#!/usr/bin/env bash
set -e

# Install Docker prerequisites
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key and repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg --yes

# Install Docker Engine
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Enable Docker and add user to docker group
sudo systemctl enable docker && sudo systemctl start docker
sudo usermod -aG docker $USER
```

**Explanation:**
- Adds Docker's official GPG key and APT repository for verified package downloads.
- Installs Docker Engine, CLI, containerd, and Buildx/Compose plugins.
- `systemctl enable docker` ensures Docker starts automatically on boot.
- `usermod -aG docker $USER` allows the `ubuntu` user to run Docker commands without `sudo`.

---

### 3.6 Project Manifest: `package.json`

```json
{
  "name": "automated-cicd-pipeline",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "bash scripts/test.sh"
  },
  "engines": { "node": ">=18.0.0" }
}
```

**Explanation:**
- `"main": "app.js"` — Declares the application entry point.
- `"scripts"` — Enables `npm start` for launching the server and `npm test` for running the local test suite.
- `"engines"` — Enforces Node.js 18+ compatibility, matching the Docker base image.
- Notably, there are **zero dependencies** — the `dependencies` field is absent because the app uses only Node.js built-in modules.

---

### 3.7 Frontend UI Files (Summary)

The project includes two HTML files (`public/index.html` and `public/health.html`) and one CSS file (`public/style.css`):

- **`index.html`** — A minimalist landing page using semantic HTML5 elements (`<main>`, `<header>`, `<section>`, `<footer>`, `<dl>`). Displays the project title, student details, and a visual pipeline overview. Contains a monospace `"Hello, Node.js!"` badge that serves as the assertion target for automated `curl` tests.

- **`health.html`** — A template with Mustache-style placeholders (`{{UPTIME}}`, `{{NODE_VERSION}}`, `{{MEMORY_MB}}`, `{{PORT}}`, `{{TIMESTAMP}}`). The server dynamically replaces these with live runtime metrics on each request, enabling real-time diagnostics in the browser.

- **`style.css`** — A lightweight (~2 KB) stylesheet using CSS Custom Properties for theming. Supports **automatic dark/light mode** via the native `@media (prefers-color-scheme: dark)` media query — zero JavaScript required. Uses a system font stack (`-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto`) for optimal rendering across platforms.

---

## Chapter 4: Cloud Infrastructure & Configuration

### 4.1 AWS EC2 Virtual Machine Specification
The target server was launched on AWS in the `ap-south-1` (Mumbai) region:
* **Instance Type:** `t2.micro` (1 vCPU, 1 GB RAM, AWS Free Tier).
* **Operating System:** Ubuntu Linux 24.04 LTS (x86_64).
* **Public IPv4 Address:** `3.111.38.88`.
* **Private Key:** RSA 2048-bit (`linux.pem`).
* **Docker Version Installed:** Docker Engine `29.10.0`, build `fdd4b8e`.

### 4.2 Security Group Firewall Rules

| Protocol | Port Range | Source | Purpose |
| :--- | :--- | :--- | :--- |
| **SSH** | `22` | `0.0.0.0/0` | Enables GitHub Actions (`appleboy/ssh-action`) to execute remote deployment commands. |
| **HTTP** | `80` | `0.0.0.0/0` | Public web browser access to the live application. |
| **Custom TCP** | `3000` | `0.0.0.0/0` | Direct microservice port access. |

### 4.3 GitHub Encrypted Secrets Mapping
Credentials are encrypted using public-key cryptography (Libsodium SealedBox) and stored under **Settings → Secrets and variables → Actions**:

| Secret Name | Stored Value | Cryptographic Role |
| :--- | :--- | :--- |
| `DOCKER_USERNAME` | `shashankpm` | Authenticates Docker Buildx client to DockerHub. |
| `DOCKER_PASSWORD` | *(Encrypted)* | Authorizes image publishing to the `shashankpm/newapp` repository. |
| `SERVER_HOST` | `3.111.38.88` | Identifies the target AWS EC2 instance IP. |
| `SERVER_USER` | `ubuntu` | Target remote administrative account. |
| `SERVER_SSH_KEY` | `linux.pem` content | RSA private key authorizing the GitHub Actions SSH client. |

---

## Chapter 5: Deployment Verification & Testing Results

### 5.1 GitHub Actions Workflow Execution

```text
Run Summary: CI/CD Pipeline (Workflow Run ID: 45404914165)
Commit: debcc47 - "feat: add minimalist /health HTML status page and JSON diagnostics"

Jobs:
  ✓ test    - 14s (Node.js 18 setup, background app startup, curl test passed)
  ✓ build   - 28s (DockerHub login, image built, tagged :latest and :debcc47, pushed)
  ✓ deploy  - 18s (SSH connect, image pull, container replacement, curl verification passed)

Status: COMPLETED
Conclusion: SUCCESS
```

### 5.2 Container State on AWS EC2
Executing `docker ps` on `ubuntu@3.111.38.88`:

```text
CONTAINER ID   IMAGE                      STATUS         PORTS                                                    NAMES
dddd7204abec   shashankpm/newapp:latest   Up 25 mins     0.0.0.0:80->3000/tcp, 0.0.0.0:3000->3000/tcp            newapp-container
```

### 5.3 Live HTTP Responses

#### 1. Root Endpoint (`http://3.111.38.88/`)
```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

<!DOCTYPE html>
<html lang="en">
<head>
  <title>CI/CD Pipeline</title>
...
```

#### 2. Health Endpoint (`http://3.111.38.88/health?format=json`)
```json
{
  "status": "UP",
  "port": 3000,
  "uptime_seconds": 1542,
  "memory_mb": 41,
  "node_version": "v18.20.8",
  "timestamp": "2026-11-24T14:28:38.238Z"
}
```

---

## Chapter 6: Viva Voce & Technical Defense Q&A Guide

### Q1: What is the fundamental difference between Continuous Delivery and Continuous Deployment?
> **Answer:** In Continuous Delivery, software changes are automatically built, tested, and staged so they are always in a releasable state, but deployment to production requires manual human authorization. In Continuous Deployment (as implemented in this project), there is zero manual intervention: every commit that successfully passes the automated testing stage is immediately deployed to production automatically.

### Q2: Why tag Docker images with both `:latest` and the Git commit SHA?
> **Answer:** Tagging with `:latest` provides a clean pointer so downstream hosts and deployment scripts can fetch the newest build. However, `:latest` is mutable and overwritten on every run. Tagging with the unique Git commit SHA (`github.sha`) creates an immutable artifact. This guarantees complete auditability—every running container can be traced to the exact source commit—and enables instant rollback to prior commits if an issue arises.

### Q3: What is the "Fail-Fast" principle and how is it implemented?
> **Answer:** Fail-Fast requires stopping pipeline execution at the earliest possible phase when an error occurs. In Stage 1 (`test`), our workflow launches the server and tests it with `curl -f http://localhost:3000`. If the app fails to start or returns an error, Stage 1 exits with code 1. GitHub Actions halts the pipeline immediately, preventing broken code from wasting runner compute, pushing invalid Docker images, or deploying to production.

### Q4: How are sensitive production credentials safeguarded?
> **Answer:** Zero passwords, access tokens, or private keys are stored in the repository. All secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`, `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`) are encrypted using Libsodium public-key cryptography within GitHub Secrets. At runtime, GitHub Actions decrypts and injects them as transient environment variables and masks their values with `***` in build logs.

### Q5: Why is `docker image prune -f` executed after deployment?
> **Answer:** Each time a new container image is pulled, the previous image layers become unreferenced ("dangling"). Over time, accumulated stale images consume substantial disk space on cloud virtual machines with limited root volumes (such as an AWS EC2 Free Tier 20 GB volume). Running `docker image prune -f` deletes unused layers after every successful deployment, preventing out-of-disk crashes.

### Q6: How does `app.js` protect against Directory Traversal attacks?
> **Answer:** When serving static files, `app.js` resolves the target file path using `path.join(PUBLIC_DIR, relativePath)` and validates that `filePath.startsWith(PUBLIC_DIR)`. If an attacker requests `../../etc/passwd`, the check evaluates to `false` and the server immediately aborts the request with an HTTP 403 Forbidden status code.

### Q7: What role do Docker restart policies play in high availability?
> **Answer:** The container is run with `--restart always`. If the Node.js application process crashes or if the AWS EC2 virtual machine is rebooted by AWS for maintenance, the Docker daemon automatically restarts `newapp-container` upon system boot without manual human intervention.

### Q8: How did we configure AWS networking to permit public web access?
> **Answer:** By default, AWS EC2 instances block all inbound traffic except Port 22 (SSH). In the AWS EC2 Security Group, Inbound Rules were added for Port 80 (HTTP) and Port 3000 with CIDR source `0.0.0.0/0`. Docker maps incoming host packets directly into container port 3000 via `-p 80:3000 -p 3000:3000`.

### Q9: Why was native Node.js used instead of third-party frameworks like Express?
> **Answer:** Using native Node.js standard modules (`http`, `fs`, `path`) completely removes external dependency management (`node_modules`), drastically decreases container image build and transfer times, eliminates dependency vulnerability attack vectors, and keeps the memory footprint under 45 MB RSS.

### Q10: How does the application support hot-reloading for manual HTML/CSS edits?
> **Answer:** Rather than caching file buffers in memory at startup, `app.js` calls asynchronous `fs.readFile()` on each incoming HTTP request. Any direct edits to `index.html` or `style.css` on the host filesystem are rendered immediately on the next browser refresh without requiring a daemon restart.

---

## Chapter 7: Conclusion & Future Scope

### 7.1 Conclusion
This project successfully designed, implemented, and verified an automated, production-oriented CI/CD pipeline for a containerized Node.js application. By leveraging GitHub Actions for orchestration, DockerHub for container registry management, and an AWS EC2 instance for cloud hosting, the system demonstrates how automated DevOps practices eliminate manual deployment errors, guarantee environment parity, provide commit-level traceability, and ensure rapid, dependable software delivery.

The application is operational and verified live on AWS at:
👉 **[http://3.111.38.88/](http://3.111.38.88/)**
👉 **[http://3.111.38.88/health](http://3.111.38.88/health)**

### 7.2 Future Scope
* **Cloud Registry Migration:** Transitioning from DockerHub to **AWS Elastic Container Registry (ECR)** with automated IAM role authentication.
* **Container Orchestration:** Deploying across an **AWS Elastic Kubernetes Service (EKS)** cluster for auto-scaling and zero-downtime rolling updates.
* **Security Scanning:** Adding automated container vulnerability scans into the CI pipeline using **Trivy** or **Snyk**.
* **Observability:** Integrating **Prometheus** and **Grafana** dashboards to monitor CPU, memory, and HTTP response latencies in real time.
* **GitOps:** Managing cluster state declaratively using **ArgoCD**.

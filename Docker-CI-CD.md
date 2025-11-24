A production-grade **CI/CD Version Management Strategy** for a Dockerized FastAPI application. The core philosophy here is **"Build Once, Promote Anywhere"**—you build the Docker image once (guaranteeing consistency) and promote that *exact* image artifact through Dev, Staging, and Production.

### 1\. The Branch-to-Environment Strategy

Do not rebuild the image for every environment. Build it once, tag it, and move it.

| Branch | Environment | Trigger | Docker Tag Strategy |
| :--- | :--- | :--- | :--- |
| **`feature/*`** | **Local / Ephemeral** | PR Open | `pr-101` (Temporary build for testing) |
| **`develop`** | **Development** | Push/Merge | `sha-7b3f1a` (Short Git Commit SHA) |
| **`release/*`** | **Staging** | Branch Create | `rc-1.2.0` (Release Candidate) |
| **`main`** | **Production** | Tag Create | `v1.2.0` (Semantic Version) |

-----

### 2\. Workflow Automation (The "How-To")

#### A. Semantic Versioning (SemVer)

Instead of manual versioning, use **Conventional Commits** to automate this.

  * **Tool:** `python-semantic-release` or `standard-version`.
  * **Logic:**
      * `fix: ...` commit → Patches version (`1.0.0` → `1.0.1`)
      * `feat: ...` commit → Minors version (`1.0.0` → `1.1.0`)
      * `BREAKING CHANGE: ...` → Majors version (`1.0.0` → `2.0.0`)
  * **Pipeline Step:** When merging to `main`, the CI tool analyzes commit messages, calculates the new version, updates `pyproject.toml` (or `version.py`), and creates a Git tag.

#### B. Container Registry Management (Promotion)

Avoid "re-building" images for Production. Retag existing ones to ensure the code tested in Staging is *identical* to Production code.

1.  **Build Step:** CI builds image `myapp:sha-xyz`.
2.  **Dev Deploy:** Deploy `myapp:sha-xyz` to Dev server.
3.  **Promotion:** When ready for Prod, **do not run `docker build` again.**
      * Pull the Dev image: `docker pull myapp:sha-xyz`
      * Retag it: `docker tag myapp:sha-xyz myapp:v1.2.0`
      * Push: `docker push myapp:v1.2.0`

-----

### 3\. Configuration & Secrets Management

**Never** bake configs into the Docker image. The image must be environment-agnostic.

  * **Strategy:** Use **Environment Variable Injection**.
  * **Implementation:**
    1.  **FastAPI:** Use `pydantic-settings` to read from env vars.
    2.  **Docker Compose:** Use variable interpolation.

**`docker-compose.yml` (Generic):**

```yaml
services:
  api:
    image: ghcr.io/myorg/fastapi-app:${IMAGE_TAG}  # Injected by CI
    ports:
      - "${HOST_PORT}:8000"
    env_file:
      - .env  # Created dynamically on the server by CI/CD
    environment:
      - DB_URL=${DB_URL}
```

  * **Dev:** CI creates a `.env` file with dev credentials on the Dev server.
  * **Prod:** CI creates a `.env` file with prod credentials on the Prod server (fetched from GitHub Secrets/GitLab Variables).

-----

### 4\. Automated Image Cleanup

Docker registries grow fast. You need an automated reaper policy.

  * **For Cloud Registries (AWS ECR / Azure ACR / Google AR):**
      * Use **Lifecycle Policies**. Set a rule: *"Expire images older than 14 days if count \> 10, EXCEPT tags starting with `v` (production versions)."*
  * **For Docker Hub / Self-Hosted:**
      * Run a weekly scheduled CI job using a tool like **`docker-registry-pruner`** or a simple script:
    <!-- end list -->
    ```bash
    # Example: Delete untagged images (dangling)
    docker image prune -f

    # Advanced: Remove images older than X days (requires registry API scripting)
    ```

-----

### 5\. Implementation Example (GitHub Actions)

Here is a simplified standard CI/CD workflow file `pipeline.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: ["develop", "main"]
  pull_request:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 1. Test & Lint
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Pytest
        run: docker compose run api pytest

  # 2. Build & Push (Dev)
  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          push: true

  # 3. Release & Promote (Prod)
  release-prod:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Semantic Release
        uses: python-semantic-release/python-semantic-release@master
        with:
          root_options: --build-metadata

      - name: Pull Dev Image & Retag to SemVer
        run: |
          # Pull the image built from the commit SHA
          docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          # Tag with new version (e.g., v1.0.0)
          docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.RELEASE_VERSION }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.RELEASE_VERSION }}

  # 4. Deploy (SSH to Server)
  deploy:
    needs: [build-and-push, release-prod]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Deploy to Server via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            export IMAGE_TAG=${{ github.sha }} # Or version tag for prod
            export DB_URL=${{ secrets.PROD_DB_URL }}
            docker compose pull
            docker compose up -d
```
To configure environment variables for multiple deployment stages (dev, staging, production) in a Dockerized FastAPI application, you should use separate `.env` files for each stage and dynamically load them based on the deployment target. This approach, combined with the "Build Once, Promote Anywhere" strategy, ensures your Docker images remain immutable while your configuration adapts to the environment.

### 1\. The Strategy: Dedicated `.env` Files

Create three separate configuration files in your secure repository (or better yet, inject them via CI/CD secrets) to store environment-specific variables like `DATABASE_URL`, `DEBUG`, and service endpoints.

  * **.env.dev:** For local development (e.g., local DB).
  * **.env.staging:** For the staging environment (mirrors production).
  * **.env.prod:** For the live production environment.

### 2\. Implementation in FastAPI (Dynamic Loading)

Use Pydantic's `BaseSettings` to load configuration. By default, it looks for a `.env` file, but we can override this behavior or rely on Docker to mount the correct file to that location.

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    debug: bool = False
    
    class Config:
        # Pydantic will read from a file named ".env" by default
        env_file = ".env"

settings = Settings()
```

### 3\. Configuring Docker Compose for Dynamic Injection

In your `docker-compose.yml`, use the `env_file` attribute. Instead of hardcoding a filename, parameterize the path using a shell variable (e.g., `ENV_PATH`).

```yaml
# docker-compose.yml
services:
  backend:
    # Use the image built by your CI/CD pipeline
    image: ghcr.io/myorg/fastapi-app:${IMAGE_TAG}
    env_file:
      # Defaults to .env.dev if ENV_PATH is not set
      - ${ENV_PATH:-.env.dev}
    ports:
      - "8000:8000"
```

### 4\. CI/CD Integration (The "Glue")

This is where the "Build Once" strategy meets dynamic configuration. Your CI/CD pipeline handles the injection of the correct environment file during deployment.

**Example Deployment Step (Bash/CI Script):**

```bash
# For Staging Deployment
export ENV_PATH=.env.staging
export IMAGE_TAG=rc-1.2.0
docker compose up -d

# For Production Deployment
export ENV_PATH=.env.prod
export IMAGE_TAG=v1.2.0
docker compose up -d
```

### Summary of Workflow

1.  **Develop:** Developer pushes code; CI builds image `myapp:sha-123`.
2.  **Deploy Staging:** CI runner creates `.env.staging` from secrets, sets `ENV_PATH=.env.staging`, and starts the container using `myapp:sha-123`.
3.  **Promote:** Tests pass. CI retags `myapp:sha-123` to `myapp:v1.0.0`.
4.  **Deploy Production:** CI runner creates `.env.prod` from secrets, sets `ENV_PATH=.env.prod`, and starts the container using `myapp:v1.0.0`.

**Dockerfile**
```# Use official Python image
FROM python:3.14-slim

# Set work directory inside container
WORKDIR /app

# Install system deps (optional: useful for building some Python packages)
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first (for caching layers)
# COPY requirements.txt .

# Install dependencies
# RUN pip install --no-cache-dir -r requirements.txt

# Install dependencies
RUN uv sync

# Copy project files
COPY . .

# Expose FastAPI default port
EXPOSE 8000

# Start FastAPI using uvicorn
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**.dockerignore**

```
.git
.gitignore
.github/
__pycache__/
*.py[cod]
*.pyo
.pytest_cache/
.mypy_cache/
.venv/
venv/
ENV/
build/
dist/
*.egg-info/
.DS_Store
Thumbs.db

# Do not bake local environment secrets into the image
.env

# Editor/IDE
.vscode/
.idea/

# Tests and local docs (not needed at runtime)
tests/
README.md
```

**docker-compose.yml**
```
name: 
services:
  "name_here":
    build:
      context: .
      dockerfile: Dockerfile
    image: "...":latest
    container_name: "..."
    working_dir: /app
    ports:
      - "8000:8000"
    env_file:
      - .env
    # Allocate a pseudo-TTY & keep STDIN open to mirror `-it` from README example
    tty: true
    stdin_open: true
    # Use Dockerfile CMD (uvicorn ...) instead of overriding with a reload/dev command for parity with `docker run` example.
    # For a live-reload dev experience, you can create a second service or uncomment below:
    # command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Here is the breakdown of the difference between a **Dockerfile** and a **Docker Compose** file, using the "Construction" analogy to make it clear.

### The Core Difference

  * **Dockerfile** is a **Blueprint**. It describes how to **build** a single image (the application).
  * **Docker Compose** is a **Site Plan**. It describes how to **run** and **connect** one or more containers (the application + database + cache) together.

-----

### 1\. Dockerfile (The Builder)

**"How do I make the application?"**

A `Dockerfile` is a text document that contains all the commands a user could call on the command line to assemble an image. It handles the **installation** phase.

  * **Why use it?** To package your code, dependencies, and environment into a portable artifact (Image).
  * **Key Focus:** OS version, Python libraries, copying code files, building binaries.

**Example (FastAPI):**

```dockerfile
# 1. Base Image (The OS)
FROM python:3.9-slim

# 2. Set working directory
WORKDIR /app

# 3. Install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# 4. Copy source code
COPY . .

# 5. Default command to run
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

-----

### 2\. Docker Compose (The Manager)

**"How do I run the application(s) together?"**

`docker-compose.yml` is a YAML file used to define and share multi-container applications. It handles the **runtime** phase.

  * **Why use it?** rarely does an app work alone. It needs a database, a cache, or a proxy. Compose lets you start them all with one command (`docker compose up`) and creates a network so they can talk to each other.
  * **Key Focus:** Ports, Environment Variables (`.env`), Volumes (storage), Networks, and Service dependencies.

**Example (FastAPI + Postgres):**

```yaml
version: '3.8'
services:
  # Service 1: Your App
  web:
    build: .             # "Use the Dockerfile in this folder"
    ports:
      - "8000:8000"      # "Open this port to the world"
    env_file:
      - .env             # "Inject these secrets"
    depends_on:
      - db               # "Wait for DB to start first"

  # Service 2: The Database
  db:
    image: postgres:15   # "Download this pre-made image"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

-----

### Comparison Table

| Feature | Dockerfile | Docker Compose |
| :--- | :--- | :--- |
| **Primary Goal** | **Build** an Image. | **Run** and Orchestrate Containers. |
| **Scope** | Single Service (One container). | Multi-Service (App + DB + Redis). |
| **Input** | Code + Dependencies. | Images + Configs (Ports, Vols, Nets). |
| **Command** | `docker build -t myapp .` | `docker compose up -d` |
| **Analogy** | The Recipe. | The Dinner Party (Recipe + Table setting + Guests). |

### How they work together (The Workflow)

In your specific CI/CD context (from the previous turn), here is how they interact:

1.  **Development:** You write code and update the **Dockerfile**.
2.  **Local Testing:** You run `docker compose up`. Compose reads the Dockerfile, builds the image locally, spins up a Postgres DB next to it, and connects them.
3.  **CI Pipeline:** The CI runner strictly uses the **Dockerfile** to build the image (`docker build`) and push it to the registry (Docker Hub/GHCR).
4.  **Production:** You copy only the **Docker Compose** file to the server. You change the instruction from `build: .` to `image: ghcr.io/myorg/app:v1`. The server pulls the image and runs it.

# The Shipping Container — Dockerizing the Satellite API

## Project Identity

- **DIFFICULTY:** Beginner
    
- **ESTIMATED TIME:** 60 Minutes
    
- **DOMAIN:** DevOps / Cloud Engineering
    
- **TARGET ROLE(S):** ML Engineer, DevOps Engineer
    
- **SERIES:** End-to-End Satellite Image Classification on AWS (Project 3 of 4)
    

## Key Concepts

- **Containerization** — Packaging an application and its dependencies (Python, PyTorch, OS libraries) into a single unit that runs consistently on any machine.
    
- **Docker Registry (ECR)** — A centralized repository for storing and managing Docker container images, similar to how GitHub stores code.
    
- **CPU-Optimization** — Specifically installing compute-optimized libraries to reduce container size and cost (crucial for cloud deployments).
    

## Role and Skill Alignment

**Core Skills:**

- Writing Dockerfiles for ML Applications
    
- Image Optimization (Slug Size Management)
    
- AWS Elastic Container Registry (ECR) Management
    
- Local Container Debugging
    

**Supporting Skills:**

- CLI Authentication (AWS -> Docker)
    
- Port Mapping
    

**Framework Principles Applied**

- **FW-02 DevOps Principles:** Implements "Environment Standardization" by using Docker. The container ensures the model runs exactly the same on your laptop as it will on AWS Fargate.
    
    - **Verification:** `docker run` output matches the local Python execution from Project 2.
        
- **FW-19 Secure Software:** Implements "Dependency Hygiene" by pinning specific base images (`python:3.9-slim`) and explicit dependency versions to prevent upstream breaking changes.
    
    - **Verification:** Inspection of Dockerfile showing pinned versions.
        

## ⚡ 30-Second Summary

Your API works on your machine, but "it works on my machine" isn't good enough for production. In this project, you will package your FastAPI application, the PyTorch model, and all dependencies into a Docker container. You will optimize this image for CPU inference to keep it lightweight, build it locally, and then push it to AWS Elastic Container Registry (ECR). This creates the immutable artifact that we will deploy to the cloud in the final project.

## What You’ll Build

- A Dockerfile optimized for PyTorch inference.
    
- A local Docker image of your satellite classifier.
    
- A private repository in AWS ECR containing your image.
    

## By the End of This Project, You’ll Have

- [ ] A validated Dockerfile.
    
- [ ] A running container accepting requests on localhost.
    
- [ ] An image URI pointing to your artifact in AWS ECR.
    

## Default Tooling and Environment

- **Container Engine:** Docker Desktop
    
- **Cloud Registry:** AWS ECR
    
- **CLI:** AWS CLI v2
    

## Cost Awareness

- **Risk:** Docker images for Deep Learning can be huge (>2GB), which creates storage costs and slow deployments.
    
- **Mitigation:** We will install the CPU-only version of PyTorch and avoid unnecessary build context to keep the image size optimized.
    

## Project Overview

We are simulating the "Build & Package" phase of the deployment lifecycle. You are taking the raw code and model artifacts and creating a "shippable unit." This is critical because it decouples the application from the underlying infrastructure—AWS Fargate doesn't need to know Python or PyTorch exists; it just needs to know how to run a Docker container.

## Step 0 — Before You Start

**What are we building?** A Docker image that encapsulates our API and model.

**What will exist at the end?**

`build/Dockerfile`

`execution/evidence/docker_build_log.txt`

An image stored in AWS ECR.

### Prerequisites

- Project 2 completed (Code in `build/app/`).
    
- Docker Desktop installed and running.
    
- AWS CLI configured.
    

### Initiative Skeleton Check

Ensure your build directory contains the app from Project 2:

`initiatives/satellite-classification/`

└── `build/`

├── `app/`

│ ├── `main.py`

│ ├── `model.pt`

│ └── `requirements.txt`

└──

## Step 1 — The Dockerfile Strategy

We need to tell Docker how to build our computer-in-a-box. We will use a slim version of Python to save space and explicitly install CPU-only PyTorch.

Navigate to the build directory:

`cd initiatives/satellite-classification/build`

Create the Dockerfile: Create a file named `Dockerfile` (no extension) inside the build directory.

`build/Dockerfile`

```
FROM python:3.9-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install minimal system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency definitions first (better layer caching)
COPY app/requirements.txt /app/requirements.txt

# Install CPU-only PyTorch explicitly
RUN pip install --upgrade pip && \
    pip install --no-cache-dir --index-url [https://download.pytorch.org/whl/cpu](https://download.pytorch.org/whl/cpu) \
    torch==2.3.1 torchvision==0.18.1 && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code and model
COPY app/ /app/app/

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Technical Note:**

The line `--index-url https://download.pytorch.org/whl/cpu` is critical. Without it, pip may download a CUDA-enabled build of PyTorch, dramatically increasing image size.

## Step 2 — Building the Image

Now we execute the build instructions.

Run Docker Build (capture logs):

`mkdir -p ../execution/evidence`

`docker build -t satellite-api:v1 . | tee ../execution/evidence/docker_build_log.txt`

Verify Image Creation:

`docker images | grep satellite-api`

### Validation Checkpoint

- **Output:** You should see `satellite-api` with tag `v1` in the list.
    
- **Size Check:** Look at the `SIZE` column. It should be roughly 500MB - 800MB. If it is >1.5GB, you likely downloaded the GPU version of PyTorch (check your Dockerfile).
    

**Common Mistakes & Fixes**

- **Context Error:** If Docker complains it cannot find files, ensure you are running the command from `initiatives/satellite-classification/build`.
    
- **Space Issues:** Docker builds consume disk space. If it fails, prune unused images using:
    
    `docker system prune`
    

## Step 3 — Local Container Test

Before we push to the cloud, we must prove it works locally.

Run the Container: We map port 8000 on our machine to port 8000 inside the container.

`docker run --rm --name sat-test -p 8000:8000 satellite-api:v1`

Verify Running State:

Open a second terminal and run:

`docker ps | grep sat-test`

Ensure status is "Up". If it says "Exited", run:

`docker logs sat-test`

Test the Endpoint: Use curl.

`curl -s http://127.0.0.1:8000/`

`curl -s -X POST "http://127.0.0.1:8000/predict" -F "file=@testdata/test_image.jpg;type=image/jpeg"`

(Note: Ensure you have `test_image.jpg` available. If you deleted it, re-create it as in Project 2, or copy one from your dataset if you still have it.)

### Validation Checkpoint

- **Response:** You should receive JSON like `{"prediction": "Forest", "confidence": …}`.
    
- **Success:** This proves the code, model, and dependencies are correctly packaged.
    

**Stop the Container:**

Press `Ctrl+C` in the terminal running the container.

## Step 4 — Publishing to ECR

Now we upload our verified artifact to AWS.

Set AWS variables:

`export AWS_REGION="$(aws configure get region)"`

`export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"`

`export REPO_NAME="satellite-api"`

Create the Repository:

`aws ecr create-repository --repository-name "$REPO_NAME"`

Copy the `repositoryUri` from the output (example format):

`123456789012.dkr.ecr.us-east-1.amazonaws.com/satellite-api`

Login to ECR:

`aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"`

Tag the Image:

`export ECR_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME"`

`docker tag satellite-api:v1 "$ECR_URI:v1"`

Push the Image:

`docker push "$ECR_URI:v1"`

### Validation Checkpoint

- **Console Output:** You will see progress bars for each layer being pushed.
    
- **AWS Console:** Go to ECR, open `satellite-api`. You should see the image tagged `v1`.
    

**Common Mistakes & Fixes**

- **Denied:** If you get access denied, ensure your IAM principal has ECR permissions (e.g., `AmazonEC2ContainerRegistryFullAccess`).
    
- **Repo Already Exists:** If the repo exists, the create call will fail; you can ignore and proceed.
    

## Step 5 — Secret Mission: The .dockerignore Optimization

Docker sends everything in the current directory to the build engine. Large/random files slow down builds and risk leaking secrets.

Create `.dockerignore` in the build directory.

`build/.dockerignore`

```
__pycache__/
*.pyc
*.pyo
*.pyd

.venv/
.git/
.gitignore

# Large local data not needed in the image
data/
../build/data/
../execution/
../execution/evidence/

# OS/editor junk
.DS_Store
.vscode/
.idea/
```

Rebuild: Run the build command again as v2.

`docker build -t satellite-api:v2 .`

Compare:

`docker images | grep satellite-api`

## Cleanup Phase

**Local Cleanup:** You can remove local images to save disk space (keep the ECR one).

`docker rmi satellite-api:v1 satellite-api:v2`

Do NOT delete the ECR repository. We need this for Project 4.

## Mission Accomplished — Reflection

**What we built:** A portable, immutable container image of our ML inference service. It is optimized for CPU usage and stored in a private cloud registry.

**What we validated:**

- The application runs inside a Linux container, independent of the host OS.
    
- The image size is optimized for cost and speed.
    
- The artifact is successfully pushed to AWS ECR.
    

**What to improve next:** In the final project, we will use AWS ECS Fargate to pull this image and run it as a production API.

## Series Continuity Note

- **Save:** The ECR Repository URI. You will absolutely need this string for Project 4. Write it down or save it in a text file in `execution/notes/`.
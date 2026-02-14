# Series Identity

- **Series Title:** End-to-End Satellite Image Classification on AWS
    
- **Canonical Task Sentence:** Build, containerize, and deploy a PyTorch satellite classifier behind a FastAPI inference API on AWS using free-tier resources.
    
- **Domain:** Machine Learning Engineering / Cloud Engineering
    
- **Target Audience:** Beginner ML Engineers seeking deployment skills.
    

# Difficulty & Scope Lock

- **Skill Level:** Beginner (SRC-0.2 Level 1: Happy-path implementations, conceptual understanding).
    
- **Project Count:** 4 Projects (Fixed).
    
- **Estimated Time:** ~60 minutes per project (4 hours total).
    
- **Cost Target:** Near-Zero (AWS Free Tier aligned).
    

# Role & Skill Alignment

- **Primary Role:** Machine Learning Engineer
    
- **Secondary Role:** Cloud Engineer
    
- **Core Skills:**
    
    - Model Serving: Exposing PyTorch models via REST APIs (FastAPI).
        
    - Containerization: Packaging ML applications for portability (Docker).
        
    - Cloud Deployment: Deploying containers to managed services (AWS ECS Fargate).
        
    - Artifact Management: Managing container images (AWS ECR) and model weights (AWS S3).
        

# Business Context

- **Problem:** Satellite imagery analysis is manually intensive and does not scale for time-sensitive workflows such as disaster response or environmental monitoring. Manual review introduces latency and operational bottlenecks.
    
- **Solution:** An automated, containerized inference API that classifies earth observation images on-demand using cloud-native infrastructure.
    
- **Value:** Enables rapid, scalable analysis of remote sensing data without managing servers. Using AWS Fargate removes host management overhead while maintaining predictable cost.
    

# Tooling & Environment Strategy

- **Local Development:** VS Code or Cursor, Python 3.12+, Docker Desktop
    
- **ML Framework:** PyTorch (CPU-only builds for cost control and image size reduction)
    
- **API Framework:** FastAPI
    
- **Cloud Provider:** AWS
    
    - S3 (Model artifact storage)
        
    - ECR (Container registry)
        
    - ECS Fargate (Serverless containers)
        
    - CloudWatch (Logging)
        
- **Data Source:** EuroSAT (RGB) public dataset, 10 land-cover classes
    

# Series Roadmap

## Project 1: The Model Factory

- **Goal:** Train a lightweight satellite classifier and persist artifacts to the cloud.
    
- **Key Actions:**
    
    - Fetch EuroSAT data
        
    - Train a ResNet-based model (CPU-only)
        
    - Evaluate validation accuracy
        
    - Upload model.pt to AWS S3
        
- **Output:** Saved model artifact in S3
    

## Project 2: The Inference Engine

- **Goal:** Wrap the model in a high-performance web API.
    
- **Key Actions:**
    
    - Build FastAPI structure
        
    - Implement /predict endpoint
        
    - Perform image preprocessing consistent with training
        
    - Integration test locally
        
- **Output:** Functional local API returning JSON predictions
    

## Project 3: The Shipping Container

- **Goal:** Package the application for portable deployment.
    
- **Key Actions:**
    
    - Write Dockerfile
        
    - Optimize image size (CPU-only PyTorch, minimal base image)
        
    - Build local image
        
    - Authenticate to AWS ECR
        
    - Push container image
        
- **Output:** Image URI stored in AWS ECR
    

## Project 4: Launch Control

- **Goal:** Deploy the API publicly and validate runtime behavior.
    
- **Key Actions:**
    
    - Create ECS Task Definition
        
    - Configure Fargate Service in public subnet
        
    - Deploy task
        
    - Validate via curl
        
    - Inspect logs in CloudWatch
        
    - Perform mandatory teardown
        
- **Output:** Public API endpoint successfully serving predictions
    

# Cost & Safety Constraints

- **Compute:**
    
    - CPU-only training (local)
        
    - Fargate 0.25 vCPU / 0.5GB memory
        
    - No GPUs
        
- **Storage:**
    
    - S3 Standard, small artifact (<100MB)
        
    - Dataset stored locally, not in S3
        
- **Networking:**
    
    - Default VPC
        
    - Public subnet for simplicity
        
    - Single inbound rule (HTTP port 80)
        
- **Teardown:**
    
    - Mandatory deletion of ECS Service
        
    - Delete Cluster
        
    - Delete Security Group
        
    - Delete Log Group
        
    - Optional: delete ECR repository and S3 bucket
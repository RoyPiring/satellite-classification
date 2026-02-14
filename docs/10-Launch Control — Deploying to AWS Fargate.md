# Launch Control — Deploying to AWS Fargate

## Project Identity

- **DIFFICULTY:** Beginner
    
- **ESTIMATED TIME:** 60 Minutes
    
- **DOMAIN:** Cloud Engineering / DevOps
    
- **TARGET ROLE(S):** Cloud Engineer, ML Engineer
    
- **SERIES:** End-to-End Satellite Image Classification on AWS (Project 4 of 4)
    

## Key Concepts

- **Serverless Containers (Fargate)** — A compute engine for containers that removes the need to manage the underlying servers (EC2). You just say "run this container," and AWS finds a place for it.
    
- **Task Definition** — The blueprint for your application. It tells AWS which Docker image to use, how much CPU/RAM it needs, and which ports to open.
    
- **Security Groups** — A virtual firewall for your container. We must explicitly allow traffic on Port 80, or the internet will be blocked from reaching our API.
    

## Role and Skill Alignment

**Core Skills:**

- AWS ECS (Elastic Container Service) Management
    
- Networking Fundamentals (VPC, Subnets, Security Groups)
    
- Infrastructure Configuration (JSON-based definitions)
    
- Log Analysis (CloudWatch)
    

**Supporting Skills:**

- API Integration Testing
    
- Resource Teardown & Cost Management
    

**Framework Principles Applied**

- **FW-16 Cloud Security Architecture:** We implement "Least Privilege Network Access" by creating a dedicated Security Group that only allows inbound HTTP traffic on Port 80, blocking all other access.
    
    - **Verification:** `aws ec2 describe-security-groups` output showing exactly one ingress rule.
        
- **FW-09 Observability:** We integrate "Golden Signal: Logs" by configuring the Task Definition to send application output (stdout) directly to AWS CloudWatch Logs.
    
    - **Verification:** Viewing the API startup logs in the CloudWatch console.
        

## ⚡ 30-Second Summary

You have a Docker image sitting in ECR (from Project 3), but it's doing nothing. In this project, you will create an ECS Task Definition to define how your container should run. You will then launch it using AWS Fargate, making your satellite classifier accessible to the world via a public IP address. Finally, you will perform a full system test and then execute a complete teardown to leave your AWS account clean.

## What You’ll Build

- A `task-definition.json` configuration file.
    
- A running serverless service on AWS ECS Fargate.
    
- A public API endpoint serving predictions.
    

## By the End of This Project, You’ll Have

- [ ] A dedicated Security Group for your API.
    
- [ ] A registered ECS Task Definition.
    
- [ ] A live Fargate Service running your container.
    
- [ ] Successful classification of satellite imagery via the public internet.
    
- [ ] A clean AWS account (post-teardown).
    

## Default Tooling and Environment

- **CLI:** AWS CLI v2
    
- **Service:** AWS ECS (Fargate Launch Type)
    
- **Logging:** AWS CloudWatch
    
- **Networking:** Default VPC (Public Subnets)
    

## Cost Awareness

- **Critical:** Fargate costs money for every second it runs.
    
- **Safety:** We use the smallest supported size (0.25 vCPU, 0.5GB RAM).
    
- **Action:** You MUST complete the Cleanup Phase immediately after verification.
    

## Project Overview

We are simulating the "Release & Operate" phase. You are taking the packaged artifact and provisioning runtime infrastructure to host it. We will use the AWS CLI to keep our deployment repeatable and documented.

## Step 0 — Before You Start

**What are we building?** JSON configurations to tell AWS how to run our Docker container, and CLI commands to launch it.

**What will exist at the end?**

`build/configs/task-def.json`

`execution/evidence/public_endpoint_test.json`

A live (then deleted) ECS Service.

### Prerequisites

- Project 3 completed (You need the ECR URI).
    
- AWS CLI configured with sufficient permissions.
    

### Initiative Skeleton Check

Create configs folder if missing:

`mkdir -p initiatives/satellite-classification/build/configs`

## Step 1 — Network Security

We will use the Default VPC and create a dedicated Security Group for our API.

Set region variable:

`export AWS_REGION="$(aws configure get region)"`

Get Default VPC ID:

`export VPC_ID="$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)"`

`echo $VPC_ID`

Create a Security Group:

`export SG_ID="$(aws ec2 create-security-group --group-name satellite-api-sg --description 'Security group for Satellite API (HTTP only)' --vpc-id $VPC_ID --query 'GroupId' --output text)"`

`echo $SG_ID`

Allow Port 80 (public HTTP access):

`aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0`

### Validation Checkpoint

`aws ec2 describe-security-groups --group-ids $SG_ID`

**Success Criteria:**

- `IpPermissions` shows exactly one rule allowing tcp port 80 from 0.0.0.0/0
    

## Step 2 — Defining the Task

Set required variables:

`export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"`

`export REPO_NAME="satellite-api"`

`export ECR_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$REPO_NAME:v1"`

Create CloudWatch Log Group:

`aws logs create-log-group --log-group-name /ecs/satellite-api --region $AWS_REGION 2>/dev/null || true`

Ensure `ecsTaskExecutionRole` exists (create if missing):

```
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ecs-tasks.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

`aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust-policy.json 2>/dev/null || true`

`aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy`

Create `build/configs/task-def.json`:

```
cat > initiatives/satellite-classification/build/configs/task-def.json <<EOF
{
  "family": "satellite-api-task",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "satellite-api",
      "image": "$ECR_URI",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "command": [
        "uvicorn",
        "app.main:app",
        "--host",
        "0.0.0.0",
        "--port",
        "80"
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/satellite-api",
          "awslogs-region": "$AWS_REGION",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF
```

Register the Task Definition:

`aws ecs register-task-definition --cli-input-json file://initiatives/satellite-classification/build/configs/task-def.json`

### Validation Checkpoint

The output JSON should show:

`"status": "ACTIVE"`

## Step 3 — Launching the Service

Create an ECS Cluster:

`aws ecs create-cluster --cluster-name satellite-cluster`

Get Default public subnets (Default VPC):

`export SUBNET_IDS="$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true --query 'Subnets[].SubnetId' --output text)"`

`echo $SUBNET_IDS`

Pick the first subnet for simplicity:

`export SUBNET_ID="$(echo $SUBNET_IDS | awk '{print $1}')"`

`echo $SUBNET_ID`

Create the Service (run 1 task with a public IP):

`aws ecs create-service --cluster satellite-cluster --service-name satellite-service --task-definition satellite-api-task --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"`

Wait for the service to become stable:

`aws ecs wait services-stable --cluster satellite-cluster --services satellite-service`

### Validation Checkpoint

- In AWS Console -> ECS -> Clusters -> satellite-cluster -> Services, `Last Status` should be `RUNNING`.
    

## Step 4 — Verification and Logs

Find the running task ARN:

`export TASK_ARN="$(aws ecs list-tasks --cluster satellite-cluster --service-name satellite-service --query 'taskArns[0]' --output text)"`

`echo $TASK_ARN`

Get the ENI ID attached to the task:

`export ENI_ID="$(aws ecs describe-tasks --cluster satellite-cluster --tasks $TASK_ARN --query 'tasks[0].attachments[0].details[?name==networkInterfaceId].value | [0]' --output text)"`

`echo $ENI_ID`

Get the Public IP:

`export PUBLIC_IP="$(aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID --query 'NetworkInterfaces[0].Association.PublicIp' --output text)"`

`echo $PUBLIC_IP`

Test Connectivity:

`curl -s "http://$PUBLIC_IP/"`

`curl -s "http://$PUBLIC_IP/docs"`

Run a Live Prediction (ensure you have an image available locally):

`curl -s -X POST "http://$PUBLIC_IP/predict" -F "file=@initiatives/satellite-classification/build/testdata/test_image.jpg;type=image/jpeg" | tee initiatives/satellite-classification/execution/evidence/public_endpoint_test.json`

Check Logs:

`aws logs tail /ecs/satellite-api --follow --since 5m`

### Validation Checkpoint

- Verify `initiatives/satellite-classification/execution/evidence/public_endpoint_test.json` exists and contains JSON output.
    

## Step 5 — Secret Mission: Load Testing (Simulated)

Create a stress script `build/scripts/stress_test.py`:

`build/scripts/stress_test.py`

```
import sys
import time
import json
import requests

def main():
    if len(sys.argv) != 2:
        print("Usage: python build/scripts/stress_test.py http://<IP>")
        sys.exit(1)
    
    # Stress test logic implementation here
    pass

if __name__ == "__main__":
    main()
```

Install requests if needed:

`pip install requests==2.32.3`

Run it:

`python initiatives/satellite-classification/build/scripts/stress_test.py http://$PUBLIC_IP`

Observe:

Check CloudWatch logs again. Do you see the spike in requests?

## Cleanup Phase (MANDATORY)

⚠️ **WARNING:** Leaving Fargate tasks running will incur costs.

Stop the Service:

`aws ecs update-service --cluster satellite-cluster --service satellite-service --desired-count 0`

`aws ecs delete-service --cluster satellite-cluster --service satellite-service --force`

`aws ecs wait services-inactive --cluster satellite-cluster --services satellite-service`

Deregister Task Definition (optional cleanup):

`export TASK_DEF_ARN="$(aws ecs describe-task-definition --task-definition satellite-api-task --query 'taskDefinition.taskDefinitionArn' --output text)"`

`aws ecs deregister-task-definition --task-definition $TASK_DEF_ARN`

Delete Cluster:

`aws ecs delete-cluster --cluster satellite-cluster`

Delete Security Group:

`aws ec2 delete-security-group --group-id $SG_ID`

Delete Log Group:

`aws logs delete-log-group --log-group-name /ecs/satellite-api`

(Optional) Delete ECR Repo and S3 Bucket if you are fully done with the series.

## Mission Accomplished — Reflection

**What we built:** A production-ready cloud deployment. We moved from local training (Proj 1), to a local API (Proj 2), to a container (Proj 3), and finally to a publicly accessible serverless API (Proj 4).

**What we validated:**

- Our container works in the cloud environment.
    
- Security groups firewall our application.
    
- We can monitor via CloudWatch.
    

**What to improve next:**

- **CI/CD:** Automate build -> push -> deploy using CodePipeline.
    
- **HTTPS:** Add an ALB + Route53 for TLS.
    

## Series Continuity Note

This concludes the End-to-End Satellite Image Classification on AWS series.

- **Final Output:** A comprehensive repository containing training scripts, API code, Docker configs, and deployment definitions.
    
- **Next Steps:** Generate final diagrams and documentation for this series.
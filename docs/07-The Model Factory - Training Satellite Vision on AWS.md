# The Model Factory - Training Satellite Vision on AWS

## Project Identity

- **DIFFICY:** Beginner
    
- **ESTIMATED TIME:** 60 Minutes
    
- **DOMAIN:** Machine Learning Engineering
    
- **TARGET ROLE(S):** Machine Learning Engineer, Cloud Engineer
    
- **SERIES:** End-to-End Satellite Image Classification on AWS (Project 1 of 4)
    

## Key Concepts

- **Transfer Learning** — Adapting a pre-trained model (ResNet) to a new task (Satellite imagery) to save compute and time.
    
- **Model Serialization** — The process of saving a trained machine learning model’s state (weights and architecture) to a file for later use.
    
- **Object Storage (S3)** — A scalable cloud storage service used here to persist model artifacts as the "source of truth" for downstream deployments.
    

## Role and Skill Alignment

**Core Skills:**

- PyTorch Model Training (CPU-optimized)
    
- Data Ingestion & Preprocessing
    
- AWS SDK for Python (Boto3)
    

**Supporting Skills:**

- Python Environment Management
    
- Cloud Asset Management
    

## Framework Principles Applied

- **FW-25 MLOps Lifecycle:** Implements the "Model Artifact Management" principle by saving versioned model weights to a centralized store (S3).
    
    - Verification: `aws s3 ls` output showing the timestamped model file.
        
- **FW-21 Data Engineering Lifecycle:** Implements the "Ingest → Store" pattern for raw model outputs.
    
    - Verification: Local file existence check followed by remote S3 existence check.
        

## ⚡ 30-Second Summary

In this project, you will build the "brain" of your satellite classification system. You will write a Python script to download the EuroSAT dataset using Torchvision, retrain a lightweight ResNet neural network to identify terrain types (like forests, rivers, and industrial areas), and serialize the trained model’s `state_dict`. Finally, you will upload this artifact to AWS S3, making it available for the API we will build in the next project. This simulates the critical handoff between an ML Scientist (who trains) and an ML Engineer (who deploys).

## What You’ll Build

- A local Python training pipeline for satellite imagery.
    
- A trained PyTorch model file (`satellite_model.pt`).
    
- An S3 bucket acting as your model registry.
    

## By the End of This Project, You’ll Have

- [ ] A validated local Python environment with PyTorch and Boto3.
    
- [ ] A local dataset of satellite images.
    
- [ ] A trained model achieving baseline accuracy on your local machine.
    
- [ ] The model artifact successfully persisted in a private AWS S3 bucket.
    

## Default Tooling and Environment

- **Language:** Python 3.9+
    
- **Frameworks:** PyTorch, Torchvision, Boto3
    
- **Infrastructure:** AWS S3 (Free Tier)
    
- **IDE:** Cursor AI (or VS Code)
    

## Cost Awareness

- **Expected Cost:** Near-zero. S3 storage for a small model (<100MB) is negligible under Free Tier limits.
    
- **Potential Risk:** Storing massive datasets in S3 or leaving buckets public.
    
- **Avoidance:** We will keep the dataset local and enforce private bucket configuration with Block Public Access enabled.
    

## Project Overview

We are simulating the "Model Development" phase of the MLOps lifecycle. Before we can serve predictions via an API (Project 2) or deploy to the cloud (Projects 3 & 4), we need a trained model artifact. You will act as the ML Engineer responsible for standardizing the training process and ensuring the final asset is securely stored in the cloud, ready for production use.

## Step 0 — Before You Start

**What are we building?** A Python-based training workflow that produces a `.pt` file and uploads it to AWS S3.

**What will exist at the end?**

- `build/scripts/train_model.py`
    
- `execution/evidence/training_logs.txt`
    
- An S3 bucket containing `satellite_model.pt`
    

### Prerequisites

- AWS CLI installed and configured with `aws configure`.
    
- Python 3.9+ installed.
    
- Git installed.
    

### Initiative Skeleton Check

Confirm your repository structure matches the SRC-1.2 standard:

`initiatives/satellite-classification/`

├── `build/`

│ ├── `configs/`

│ ├── `scripts/`

│ └── `artifacts/`

└── `execution/`

├── `execution-report.md`

├── `runbook.md`

└── `evidence/`

If these folders do not exist, create them now:

```
mkdir -p initiatives/satellite-classification/build/{configs,scripts,artifacts}
mkdir -p initiatives/satellite-classification/execution/evidence
```

## Step 1 — Environment and Data Setup

We need to set up a clean Python environment and fetch our training data. We will use the Torchvision EuroSAT dataset suitable for CPU training.

Create and activate a virtual environment:

```
cd initiatives/satellite-classification
python3 -m venv .venv
source .venv/bin/activate
```

Install dependencies: Create a file `build/requirements.txt`:

```
torch==2.3.1
torchvision==0.18.1
boto3==1.34.162
```

Run the install:

```
python -m pip install --upgrade pip
pip install -r build/requirements.txt
```

Create the Data Ingest Script: Create `build/scripts/ingest_data.py`. This script downloads EuroSAT and verifies it can be loaded.

`build/scripts/ingest_data.py`

```
from torchvision.datasets import EuroSAT
from torchvision.transforms import ToTensor

def main():
    ds = EuroSAT(root="build/data", download=True, transform=ToTensor())
    print(f"Downloaded and loaded EuroSAT successfully. Samples: {len(ds)}")

if __name__ == "__main__":
    main()
```

Create `build/scripts/verify_data.py`:

`build/scripts/verify_data.py`

```
import os
from torchvision.datasets import EuroSAT

def main():
    root = "build/data"
    ds = EuroSAT(root=root, download=False)
    classes = ds.classes
    print(f"Success! Dataset contains {len(ds)} images")
    print(f"Classes: {classes}")

    expected_path = os.path.join(root, "eurosat")
    if not os.path.isdir(expected_path):
        raise FileNotFoundError(f"Expected dataset folder not found: {expected_path}")
    print(f"Found dataset folder: {expected_path}")

if __name__ == "__main__":
    main()
```

Run the verification:

```
python build/scripts/ingest_data.py
python build/scripts/verify_data.py
```

### Validation Checkpoint

- **Output:** The terminal should print "Success! Dataset contains 27000 images" (or similar count) and list classes like Forest, River, Highway.
    
- **Evidence:** A folder named `build/data/eurosat` should exist.
    

### Common Mistakes & Fixes

- **Disk Space:** Ensure you have ~2GB of free space.
    

## Step 2 — The Training Loop

Now we write the training logic. We will use Transfer Learning with a ResNet18 model. This allows us to train a baseline model quickly on a CPU.

Create the training script: Create `build/scripts/train_model.py`.

`build/scripts/train_model.py`

```
import os
from datetime import datetime
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, random_split
from torchvision.datasets import EuroSAT
from torchvision.transforms import Compose, Resize, ToTensor, Normalize
from torchvision.models import resnet18, ResNet18_Weights

def main():
    torch.manual_seed(42)
    # [Rest of training logic follows the described pattern]

if __name__ == "__main__":
    main()
```

Execute Training:

```
python build/scripts/train_model.py | tee execution/evidence/training_logs.txt
```

### Validation Checkpoint

- **Console Output:** You should see loss values decreasing (e.g., Loss: 2.3… -> Loss: 1.1…).
    
- **File Check:** Verify `build/artifacts/satellite_model.pt` exists.
    
- **Evidence:** Verify `execution/evidence/training_logs.txt exists`.
    

### Common Mistakes & Fixes

- **Memory Error:** If the process crashes, reduce `batch_size` to 16 or 8.
    
- **Slow Training:** We explicitly limited the loop to 100 batches (`max_train_batches = 100`) to keep this tutorial project fast. Remove this line for a full production run.
    

## Step 3 — Cloud Persistence (S3)

A model on your laptop is useless to a cloud application. We must upload it to S3.

Create a unique S3 Bucket: Bucket names must be globally unique.

Set variables (example):

```
export AWS_REGION="$(aws configure get region)"
export BUCKET_NAME="satellite-model-registry-$RANDOM-$RANDOM"
```

Create the bucket (handles non-us-east-1 correctly):

```
if [ "$AWS_REGION" = "us-east-1" ]; then
  aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION"
else
  aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$AWS_REGION" --create-bucket-configuration LocationConstraint="$AWS_REGION"
fi
```

Enforce private defaults (Block Public Access + Bucket Owner Enforced):

```
aws s3api put-public-access-block \
--bucket "$BUCKET_NAME" \
--public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-ownership-controls \
--bucket "$BUCKET_NAME" \
--ownership-controls Rules=[{ObjectOwnership=BucketOwnerEnforced}]
```

Create the Upload Script: Create `build/scripts/upload_artifact.py`:

`build/scripts/upload_artifact.py`

```
import os
import sys
import boto3

def main():
    bucket = os.environ.get("BUCKET_NAME")
    if not bucket:
        print("Missing BUCKET_NAME env var.", file=sys.stderr)
        sys.exit(1)
    # [Rest of upload logic follows the described pattern]

if __name__ == "__main__":
    main()
```

Run the Upload:

```
python build/scripts/upload_artifact.py
```

### Validation Checkpoint

- Run `aws s3 ls s3://$BUCKET_NAME/models/v1/`
    
- **Success:** You should see `satellite_model.pt` listed with its size and timestamp.
    

## Step 4 — Verification and Cleanup Prep

We need to verify that what we built allows us to download and load the model, proving it works.

Create Verification Script: Create `build/scripts/verify_download.py`:

`build/scripts/verify_download.py`

```
import os
import sys
import boto3
import torch
import torch.nn as nn
from torchvision.models import resnet18, ResNet18_Weights

def build_model(num_classes: int):
    model = resnet18(weights=ResNet18_Weights.DEFAULT)
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    return model

def main():
    bucket = os.environ.get("BUCKET_NAME")
    if not bucket:
        print("Missing BUCKET_NAME env var.", file=sys.stderr)
        sys.exit(1)
    # [Rest of verification logic follows the described pattern]

if __name__ == "__main__":
    main()
```

Run Verification:

```
python build/scripts/verify_download.py
```

### Validation Checkpoint

- **Output:** "Model loaded successfully. Verification complete."
    
- This confirms our S3 artifact is not corrupt and can be used by the API in Project 2.
    

## Step 5 — Optional Secret Mission: Metadata Versioning

In production, just saving the file isn't enough. We need to know metrics about the model.

Modify the upload script to also upload a `metadata.json` side-by-side with the model.

Content of `metadata.json`:

```
{
  "model_name": "satellite_model",
  "version": "v1",
  "framework": "pytorch",
  "architecture": "resnet18",
  "dataset": "EuroSAT",
  "num_classes": 10,
  "val_accuracy": 0.0,
  "trained_at_utc": "REPLACE_WITH_VALUE_FROM_TRAINING_OUTPUT",
  "artifact_key": "models/v1/satellite_model.pt"
}
```

Upload it to `s3://$BUCKET_NAME/models/v1/metadata.json`.

**Constraint:** Do not use external MLOps tools (like MLflow) yet; stick to Boto3 and S3.

## Cleanup Phase

Since we need these artifacts for Project 2, DO NOT DELETE THE S3 BUCKET OR THE MODEL.

However, to save local disk space:

You can delete the `build/data/` folder (the downloaded images) if you are low on space. The `artifacts/` folder is small and should be kept.

## Mission Accomplished — Reflection

**What we built:** A complete model factory pipeline. We ingested raw data, transformed it, trained a neural network, and published the result to a cloud-native object store.

**What we validated:**

- Data integrity via the verification script.
    
- Training logic via the loss output.
    
- Artifact persistence via the upload and re-download verification.
    

**What to improve next:** In Project 2, we will stop running scripts manually and wrap this model in a FastAPI application to serve real-time predictions.

## Series Continuity Note

- **Keep:** The S3 Bucket Name and the `satellite_model.pt` file. You will need the bucket name for the next project.
    
- **Discard:** The local `build/data/` folder can be redownloaded if needed.
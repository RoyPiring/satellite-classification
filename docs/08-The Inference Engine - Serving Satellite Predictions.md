# The Inference Engine — Serving Satellite Predictions

## Project Identity

- **DIFFICULTY:** Beginner
    
- **ESTIMATED TIME:** 60 Minutes
    
- **DOMAIN:** Machine Learning Engineering
    
- **TARGET ROLE(S):** Machine Learning Engineer, Backend Engineer
    
- **SERIES:** End-to-End Satellite Image Classification on AWS (Project 2 of 4)
    

## Key Concepts

- **Inference** — The process of using a trained machine learning model to make predictions on new, unseen data.
    
- **REST API** — A standard architectural style for computer systems on the web, used here to allow users to send images to our model and get results back.
    
- **Deserialization** — Converting the saved model bytes (from S3) back into a functioning Python object in memory.
    

## Role and Skill Alignment

**Core Skills:**

- FastAPI Development
    
- PyTorch Inference (CPU)
    
- Image Preprocessing
    

**Supporting Skills:**

- API Testing (cURL/Python Requests)
    
- S3 Artifact Retrieval
    

**Framework Principles Applied**

- **FW-04 Software Delivery Lifecycle (SDLC) Principles:** We define a clear API contract (input: image, output: JSON class/confidence) before implementation.
    
    - **Verification:** Successful response from `/predict` matching the defined JSON schema.
        
- **FW-24 AI Integration Safety:** We implement input validation to ensure only valid image files are processed, preventing processing errors or potential exploits.
    
    - **Verification:** API error response when submitting a text file instead of an image.
        

## ⚡ 30-Second Summary

In the previous project, you trained a model and stored it in the "cold" storage of S3. Now, you will bring it to life. You will build a FastAPI application that downloads your model, loads it into memory, and exposes a `/predict` endpoint. This turns your static model file into a responsive service that can classify satellite imagery in milliseconds on your local machine.

## What You’ll Build

- A Python-based API server using FastAPI.
    
- An inference pipeline that transforms raw images into model predictions.
    
- A local integration test suite.
    

## By the End of This Project, You’ll Have

- [ ] A running local web server listening on port 8000.
    
- [ ] A functioning prediction endpoint that accepts image uploads.
    
- [ ] Proof that your model correctly identifies terrain types from new images.
    

## Default Tooling and Environment

- **Language:** Python 3.9+
    
- **Frameworks:** FastAPI, Uvicorn, PyTorch, Boto3
    
- **Interface:** HTTP (via browser or cURL)
    

## Cost Awareness

- **Expected Cost:** Near-zero. We are running the compute locally. S3 retrieval costs are negligible for this file size.
    

## Project Overview

We are simulating the "Model Serving" phase. A model file sitting in S3 provides no value until it is accessible. We will build the software layer that sits between the user (who has an image) and the model (which has the intelligence).

## Step 0 — Before You Start

**What are we building?** A `main.py` file containing the FastAPI application logic and helper scripts to run it.

**What will exist at the end?**

`build/app/main.py`

`build/app/requirements.txt`

`execution/evidence/api_response.json`

### Prerequisites

- Project 1 completed (S3 bucket exists with `satellite_model.pt`).
    
- AWS credentials configured locally.
    

### Initiative Skeleton Check

Ensure your directory looks like this (from Project 1):

`initiatives/satellite-classification/`

├── `build/`

│ ├── `scripts/`

│ └── `artifacts/`

└── `execution/`

Create the app directory:

`mkdir -p initiatives/satellite-classification/build/app`

`touch initiatives/satellite-classification/build/app/__init__.py`

## Step 1 — Artifact Retrieval

We need to simulate a production cold-start: the server boots up, reaches out to S3, and pulls the latest model weight.

Activate your environment (if not already active):

`cd initiatives/satellite-classification`

`source .venv/bin/activate`

Update dependencies: Create `build/app/requirements.txt`:

`fastapi==0.111.1`

`uvicorn[standard]==0.30.3`

`python-multipart==0.0.9`

`pillow==10.4.0`

`torch==2.3.1`

`torchvision==0.18.1`

`boto3==1.34.162`

Install them:

`pip install -r build/app/requirements.txt`

Create the Download Script: Create `build/app/download_model.py`. You will need your bucket name from Project 1.

`build/app/download_model.py`

```
import os
import sys
import boto3

def main():
    bucket = os.environ.get("BUCKET_NAME")
    if not bucket:
        print("Missing BUCKET_NAME env var.", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Run the download:

`python build/app/download_model.py`

### Validation Checkpoint

- **Verify:** Check that `build/app/model.pt` exists. This file is the heart of our application.
    

**Common Mistakes & Fixes**

- **Access Denied:** Ensure your AWS CLI is configured correctly.
    
- **Bucket Not Found:** Double-check the bucket name spelling from Project 1 using `aws s3 ls`.
    

## Step 2 — The API Skeleton

Now we build the FastAPI shell.

Create the main application file: Create `build/app/main.py`.

`build/app/main.py`

```
import os
import sys
from contextlib import asynccontextmanager
import torch
import torch.nn as nn
from fastapi import FastAPI
from torchvision.models import resnet18, ResNet18_Weights

MODEL_PATH = "build/app/model.pt"

def build_model(num_classes: int):
    model = resnet18(weights=ResNet18_Weights.DEFAULT)
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    return model

@asynccontextmanager
async def lifespan(app: FastAPI):
    print("Loading ResNet18 model…")
    if not os.path.isfile(MODEL_PATH):
        print(f"Model file not found: {MODEL_PATH}", file=sys.stderr)
        raise FileNotFoundError(MODEL_PATH)

app = FastAPI(lifespan=lifespan)

@app.get("/")
def root():
    return {"status": "online", "model_loaded": True}
```

Test the Skeleton: Run the server from the build directory context so paths align.

`cd initiatives/satellite-classification/build`

`uvicorn app.main:app --reload --host 127.0.0.1 --port 8000`

### Validation Checkpoint

- **Console Output:** You should see `Loading ResNet18 model…` followed by `Model loaded successfully.` and `Application startup complete.`
    
- **Browser:** Visit `http://127.0.0.1:8000`. You should see `{"status":"online","model_loaded":true}`.
    

**Common Mistakes & Fixes**

- **Module Not Found:** Ensure you run `uvicorn` from `initiatives/satellite-classification/build`, not `build/app`.
    
- **Path Error:** If it says file not found, ensure `download_model.py` put the file in `build/app/model.pt`.
    

## Step 3 — Inference Logic

We need to transform the user's uploaded image exactly how we transformed the training data in Project 1 (Resize, Normalize), run it through the model, and return the result.

Update `build/app/main.py`: Add the transformation logic and the predict endpoint.

`build/app/main.py`

```
import os
import sys
from contextlib import asynccontextmanager
from io import BytesIO
import torch
import torch.nn as nn
from fastapi import FastAPI, File, HTTPException, UploadFile
from PIL import Image
from torchvision.models import resnet18, ResNet18_Weights
from torchvision.transforms import Compose, Resize, ToTensor, Normalize

MODEL_PATH = "build/app/model.pt"

# Safety limits
MAX_UPLOAD_BYTES = 5 * 1024 * 1024
ALLOWED_CONTENT_TYPES = {"image/jpeg", "image/png", "image/webp"}

# Secret mission constant (used in Step 5 too)
CONFIDENCE_THRESHOLD = 0.60

def build_model(num_classes: int):
    model = resnet18(weights=ResNet18_Weights.DEFAULT)
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    return model

transform = Compose([
    Resize((224, 224)),
    ToTensor(),
    Normalize(mean=(0.485, 0.456, 0.406), std=(0.229, 0.224, 0.225)),
])

def load_image_bytes(data: bytes) -> Image.Image:
    try:
        img = Image.open(BytesIO(data))
        img.verify()  # quick integrity check
        img = Image.open(BytesIO(data)).convert("RGB")  # reopen after verify
        return img
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid image file")

@asynccontextmanager
async def lifespan(app: FastAPI):
    print("Loading ResNet18 model…")
    if not os.path.isfile(MODEL_PATH):
        print(f"Model file not found: {MODEL_PATH}", file=sys.stderr)
        raise FileNotFoundError(MODEL_PATH)

app = FastAPI(lifespan=lifespan)

@app.get("/")
def root():
    return {"status": "online", "model_loaded": True}

@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    if file.content_type not in ALLOWED_CONTENT_TYPES:
        raise HTTPException(status_code=400, detail="Unsupported file type")
```

### Validation Checkpoint

- The server should auto-reload. Ensure no syntax errors appear in the terminal.
    

## Step 4 — Testing the Inference

We will use a script to act as a client and send an image to our API.

Get a Test Image: Download a sample satellite image or use one from the data folder if you kept it. If not, download one quickly:

`mkdir -p initiatives/satellite-classification/build/testdata`

`cp initiatives/satellite-classification/build/data/eurosat/2750/AnnualCrop/AnnualCrop_1.jpg initiatives/satellite-classification/build/testdata/test_image.jpg`

If you deleted `build/data/`, re-run ingestion from Project 1 to regenerate it.

Create a Test Script: Create `build/test_api.py`:

`build/test_api.py`

```
import json
import os
import sys
import requests

def main():
    img_path = "build/testdata/test_image.jpg"
    if not os.path.isfile(img_path):
        print(f"Missing test image: {img_path}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

Install requests (local test-only dependency):

`pip install requests==2.32.3`

Run the Test: (Ensure the `uvicorn` server is still running in another terminal window).

`cd initiatives/satellite-classification`

`python build/test_api.py`

### Validation Checkpoint

- **Output:** You should see `Status Code: 200` and a JSON response like `{"prediction": "Forest", "confidence": 0.98…}`.
    
- **Evidence:** Verify `initiatives/satellite-classification/execution/evidence/api_response.json` contains the output.
    

**Common Mistakes & Fixes**

- **Connection Refused:** Make sure the `uvicorn` server is running.
    
- **400 Bad Request:** Ensure you provided a valid image file.
    

## Step 5 — Secret Mission: Confidence Thresholding

Production systems rarely accept every prediction. If the confidence is too low, the system should say "I don't know" rather than guessing.

We already added: `CONFIDENCE_THRESHOLD = 0.60`.

Test: Try uploading a non-satellite image (like a photo of a cat or dog) and see if the confidence drops and triggers the "Uncertain" label.

curl equivalent:

`curl -s -X POST "http://127.0.0.1:8000/predict" -F "file=@build/testdata/test_image.jpg;type=image/jpeg"`

## Cleanup Phase

- **Stop the Server:** Press `Ctrl+C` in the terminal running `uvicorn`.
    
- **Keep the Artifacts:** Do NOT delete `build/app/model.pt` or the `main.py` file. We need these for the Docker container in Project 3.
    
- **Delete Test Images:** You can remove `build/testdata/test_image.jpg`.
    

## Mission Accomplished — Reflection

**What we built:** A functioning ML inference service. We moved from "model training" (running a script once) to "model serving" (a continuous process listening for requests).

**What we validated:**

- Artifact retrieval from S3 works in a software context.
    
- FastAPI correctly loads the PyTorch model into memory.
    
- The API accepts binary image data and returns structured JSON predictions.
    

**What to improve next:** Currently, this only runs on your machine ("It works on my machine"). In Project 3, we will containerize this application with Docker so it can run anywhere (specifically, on AWS ECS).

## Series Continuity Note

- **Preserve:** The `build/app` directory. This entire folder will be copied into our Docker container in the next project.
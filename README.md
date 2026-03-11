# Network Security - Phishing Detection System

An end-to-end Machine Learning pipeline for detecting phishing websites using network security features. The system ingests data from MongoDB, validates and transforms it, trains classification models with hyperparameter tuning, and serves predictions via a FastAPI web application — all fully containerized and deployed with CI/CD on AWS.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Architecture](#project-architecture)
- [Pipeline Stages](#pipeline-stages)
- [API Endpoints](#api-endpoints)
- [Project Structure](#project-structure)
- [Installation & Setup](#installation--setup)
- [Usage](#usage)
- [CI/CD & Deployment](#cicd--deployment)
- [Configuration](#configuration)

---

## Features

- **Automated ML Pipeline** — Data ingestion, validation, transformation, and model training in one flow
- **Dataset Drift Detection** — Kolmogorov-Smirnov test to identify distribution shifts in features
- **Multi-Model Training** — Trains Random Forest, Gradient Boosting, Decision Tree, Logistic Regression, and AdaBoost with hyperparameter tuning
- **Best Model Selection** — Automatically selects the best model based on F1-score
- **MLflow Experiment Tracking** — Logs metrics (F1, Precision, Recall) to DAGsHub/MLflow
- **FastAPI Prediction Service** — Upload a CSV and get phishing predictions as an HTML table
- **AWS S3 Artifact Sync** — Automatically syncs artifacts and models to S3
- **Dockerized Deployment** — Containerized app with GitHub Actions CI/CD to AWS ECR

---

## Tech Stack

| Category | Technologies |
|----------|-------------|
| **ML & Data** | scikit-learn, pandas, NumPy, SciPy |
| **Web Framework** | FastAPI, Uvicorn, Jinja2 |
| **Database** | MongoDB (pymongo), certifi |
| **Experiment Tracking** | MLflow, DAGsHub |
| **Cloud & Storage** | AWS S3 (boto3), AWS ECR |
| **Infrastructure** | Docker, GitHub Actions |
| **Configuration** | PyYAML, python-dotenv |

---

## Project Architecture

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐     ┌───────────────┐
│   MongoDB    │────▶│ Data Ingestion│────▶│ Data Validation  │────▶│     Data      │
│  (Raw Data)  │     │  (80/20 split)│     │  (Drift Check)   │     │Transformation │
└──────────────┘     └───────────────┘     └──────────────────┘     │ (KNN Imputer) │
                                                                    └───────┬───────┘
                                                                            │
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐            │
│  FastAPI     │◀────│  Final Model  │◀────│  Model Trainer   │◀───────────┘
│  (Predict)   │     │  (S3 Sync)    │     │ (5 Classifiers)  │
└──────────────┘     └───────────────┘     └──────────────────┘
```

---

## Pipeline Stages

### 1. Data Ingestion
- Reads phishing data from MongoDB (`KRISHAI.NetworkData`)
- Exports feature store CSV
- Splits into **80% train / 20% test**

### 2. Data Validation
- Validates column count against schema (31 features expected)
- Runs **KS-test** (p-value threshold = 0.05) on each feature to detect dataset drift
- Generates drift report (`drift_report.yaml`)

### 3. Data Transformation
- Applies **KNN Imputer** (k=3) to handle missing values
- Converts target labels: `-1 → 0` (phishing), `1` (legitimate)
- Saves preprocessor as pickle for inference

### 4. Model Training
Trains 5 classifiers with hyperparameter tuning via GridSearchCV:

| Model | Key Hyperparameters |
|-------|-------------------|
| Random Forest | n_estimators: [8, 16, 32, 128, 256] |
| Gradient Boosting | learning_rate, subsample, n_estimators |
| Decision Tree | criterion: [gini, entropy, log_loss] |
| Logistic Regression | default |
| AdaBoost | learning_rate, n_estimators |

- Selects the **best model by F1-score** (minimum threshold: 0.6)
- Logs F1, Precision, Recall to **MLflow/DAGsHub**
- Syncs final model and artifacts to **AWS S3**

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Redirects to Swagger docs |
| `GET` | `/train` | Triggers the full training pipeline |
| `POST` | `/predict` | Upload CSV file → returns phishing predictions as HTML table |

---

## Project Structure

```
networksecurity/
├── cloud/                  # AWS S3 syncer
├── components/             # Pipeline components
│   ├── data_ingestion.py
│   ├── data_validation.py
│   ├── data_transformation.py
│   └── model_trainer.py
├── constant/               # Configuration constants
├── entity/                 # Config & artifact dataclasses
├── exception/              # Custom exception handling
├── logging/                # Structured logging
├── pipeline/               # Training & batch prediction pipelines
└── utils/
    ├── main_utils/         # YAML, pickle, numpy utilities
    └── ml_utils/
        ├── metric/         # Classification metrics (F1, Precision, Recall)
        └── model/          # NetworkModel estimator wrapper

Artifacts/                  # Timestamped pipeline outputs
final_model/                # Production-ready model & preprocessor
prediction_output/          # Inference results (output.csv)
data_schema/                # Feature schema (schema.yaml)
templates/                  # Jinja2 HTML templates
```

---

## Installation & Setup

### Prerequisites
- Python 3.10+
- MongoDB instance
- AWS account (for S3 and ECR)

### Local Setup

```bash
# Clone the repository
git clone https://github.com/Sahilpatil009/network-security.git
cd network-security

# Create virtual environment
python -m venv venv
venv\Scripts\activate      # Windows
# source venv/bin/activate # Linux/Mac

# Install dependencies
pip install -r requirements.txt

# Set environment variables
# Create a .env file with:
# MONGODB_URL_KEY=<your-mongodb-connection-string>
# AWS_ACCESS_KEY_ID=<your-key>
# AWS_SECRET_ACCESS_KEY=<your-secret>

# Push data to MongoDB (first time)
python push_data.py

# Run the application
python app.py
```

The API will be available at `http://localhost:8000/docs`

---

## Usage

### Train the Model
```bash
# Via API
curl http://localhost:8000/train

# Or run directly
python main.py
```

### Make Predictions
1. Open `http://localhost:8000/docs`
2. Use the `/predict` endpoint
3. Upload a CSV file with the 30 phishing features
4. View predictions in the returned HTML table

---

## CI/CD & Deployment

### GitHub Actions Pipeline
The project uses a 3-stage CI/CD pipeline (`.github/workflows/main.yml`):

1. **Continuous Integration** — Linting and unit tests
2. **Continuous Delivery** — Build Docker image and push to AWS ECR
3. **Continuous Deployment** — Pull and run on self-hosted EC2 runner

### GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `AWS_REGION` | AWS region (e.g., `us-east-1`) |
| `AWS_ECR_LOGIN_URI` | ECR login URI |
| `ECR_REPOSITORY_NAME` | ECR repository name |

### Docker Setup on EC2

```bash
# Update packages
sudo apt-get update -y
sudo apt-get upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker
```

---

## Configuration

| Parameter | Value |
|-----------|-------|
| Train/Test Split | 80/20 |
| KNN Imputer Neighbors | 3 |
| Min Model Accuracy | 0.6 (60%) |
| Overfitting Threshold | 0.05 |
| Drift Detection Threshold | p < 0.05 |
| MongoDB Database | `KRISHAI` |
| MongoDB Collection | `NetworkData` |
| S3 Bucket | `netwworksecurity` |
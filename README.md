# Network Security — Phishing Website Detection

An end-to-end machine learning system that detects phishing websites from URL/HTML-derived features. It covers the full ML lifecycle — data ingestion from MongoDB, schema validation with data-drift detection, data transformation, model training with hyperparameter tuning and MLflow experiment tracking — and serves the trained model through a FastAPI web app, containerized with Docker and deployed via a GitHub Actions CI/CD pipeline to AWS (ECR + EC2).

## Overview

Phishing websites try to imitate legitimate ones to steal sensitive information. This project trains a binary classifier that labels a website as **phishing** or **legitimate** based on 30 structural and behavioral features extracted from the page/URL (IP address usage, URL length, use of `@` symbol, SSL state, domain age, presence of iframes/pop-ups, etc.). The target column `Result` holds the label.

The project follows a modular, production-style ML pipeline architecture (ingestion → validation → transformation → training) rather than a single notebook script, so each stage can be run, tested, and deployed independently.

## Features

- **Data Ingestion** — Pulls raw records from a MongoDB Atlas collection, exports them as a feature store CSV, and splits them into train/test sets.
- **Data Validation** — Validates the incoming data against a defined [schema](data_schema/schema.yaml) (column count/types) and detects data drift between train and test distributions using the Kolmogorov–Smirnov test, producing a drift report.
- **Data Transformation** — Handles missing values with a KNN imputer and serializes the fitted preprocessing pipeline.
- **Model Training** — Trains and compares several classifiers (Random Forest, Decision Tree, Gradient Boosting, Logistic Regression, AdaBoost) with `GridSearchCV` hyperparameter tuning, selects the best-performing model, and logs metrics/artifacts to **MLflow** (via DagsHub).
- **Artifacts** — Every pipeline run is versioned in a timestamped `Artifacts/` directory; the final serialized model and preprocessor are saved to `final_model/`.
- **Batch Prediction API** — A FastAPI service exposes:
  - `GET /train` — runs the full training pipeline on demand.
  - `POST /predict` — accepts a CSV upload, runs inference with the saved model, and returns the predictions as an HTML table (also saved to `prediction_output/output.csv`).
- **Cloud Sync** — Utility to sync artifacts/models to an S3 bucket.
- **CI/CD** — GitHub Actions workflow that lints, builds a Docker image, pushes it to Amazon ECR, and deploys it to an EC2 instance via a self-hosted runner.

## Tech Stack

| Layer | Tools |
|---|---|
| Language | Python 3.10+ |
| Data & ML | pandas, numpy, scikit-learn |
| Database | MongoDB (Atlas) |
| Experiment Tracking | MLflow, DagsHub |
| API | FastAPI, Uvicorn |
| Containerization | Docker |
| CI/CD & Cloud | GitHub Actions, AWS ECR, AWS EC2, AWS S3 |

## Project Structure

```
networksecurity/
├── app.py                          # FastAPI app: /train and /predict endpoints
├── main.py                         # Runs the training pipeline end-to-end (script entry point)
├── push_data.py                    # Pushes the raw CSV dataset into MongoDB
├── networksecurity/
│   ├── components/                 # Pipeline stages
│   │   ├── data_ingestion.py
│   │   ├── data_validation.py
│   │   ├── data_transformation.py
│   │   └── model_trainer.py
│   ├── pipeline/
│   │   ├── training_pipeline.py    # Orchestrates all components
│   │   └── batch_prediction.py
│   ├── entity/                     # Config & artifact dataclasses
│   ├── constant/training_pipeline/ # Central pipeline constants
│   ├── utils/                      # I/O, model evaluation, metrics helpers
│   ├── cloud/s3_syncer.py          # Sync artifacts/models to S3
│   ├── exception/                  # Custom exception handling
│   └── logging/                    # Custom logger
├── data_schema/schema.yaml         # Expected column names/types
├── Network_Data/phisingData.csv    # Raw phishing dataset
├── final_model/                    # Serialized best model + preprocessor
├── Artifacts/                      # Timestamped outputs of each pipeline run
├── templates/table.html            # Jinja2 template for prediction results
├── Dockerfile
└── .github/workflows/main.yml      # CI/CD pipeline
```

## Getting Started

### Prerequisites

- Python 3.10+
- A MongoDB Atlas connection string
- (Optional) AWS account for S3 sync / ECR-EC2 deployment
- (Optional) DagsHub/MLflow account for experiment tracking

### Installation

```bash
git clone <repo-url>
cd networksecurity
pip install -r requirements.txt
```

### Configuration

Create a `.env` file in the project root:

```
MONGODB_URL_KEY=<your-mongodb-connection-string>
MONGO_DB_URL=<your-mongodb-connection-string>
```

> **Note:** `model_trainer.py` currently sets MLflow/DagsHub tracking credentials directly in code. Move these to environment variables before pushing this repository publicly, since they are currently hardcoded.

### Load the dataset into MongoDB

```bash
python push_data.py
```

### Run the training pipeline

```bash
python main.py
```

This runs data ingestion → validation → transformation → model training, and writes artifacts to `Artifacts/<timestamp>/` plus the final model to `final_model/`.

### Run the API server

```bash
python app.py
```

The app starts at `http://localhost:8000`:
- `http://localhost:8000/docs` — Swagger UI
- `GET /train` — trigger training via HTTP
- `POST /predict` — upload a CSV of feature rows to get phishing/legitimate predictions

### Run with Docker

```bash
docker build -t networksecurity .
docker run -p 8080:8080 networksecurity
```

## CI/CD Pipeline

Pushes to `main` trigger a GitHub Actions workflow ([.github/workflows/main.yml](.github/workflows/main.yml)) that:

1. **Continuous Integration** — checks out code, lints, runs tests.
2. **Continuous Delivery** — builds a Docker image and pushes it to Amazon ECR.
3. **Continuous Deployment** — a self-hosted runner (e.g., on EC2) pulls the latest image and runs it as a container.

Required GitHub Secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_ECR_LOGIN_URI`, `ECR_REPOSITORY_NAME`.

## Dataset

The dataset (`Network_Data/phisingData.csv`) contains 30 features describing website/URL characteristics (e.g., `having_IP_Address`, `URL_Length`, `SSLfinal_State`, `age_of_domain`, `Google_Index`) and a `Result` column indicating whether the site is phishing or legitimate.

## Acknowledgements

Built as a hands-on, production-style ML project covering ingestion, validation, transformation, training, experiment tracking, packaging, and cloud deployment.

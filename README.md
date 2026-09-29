# NetSentry: AI-Assisted Network Intrusion Detection Platform

A full-stack intrusion detection platform that combines a machine learning traffic classifier with a signature-based rule layer to produce a single, explainable threat score for every network flow. Detected threats are streamed live to a React dashboard.

Built with React, FastAPI, PostgreSQL, Redis, TensorFlow / scikit-learn, and Docker.

---

## Overview

Rule-based detection catches known attack patterns but misses anything new. A pure ML classifier can catch unusual behavior but is hard to explain and can be fooled by traffic that looks unlike its training data. NetSentry uses both:

- A **signature rule layer** flags well-understood attacks (port scans, SYN floods, brute-force attempts) and always says which rule fired.
- An **ML classifier** trained on CIC-IDS2017 scores each flow's likelihood of being malicious and reports the top contributing features.
- A **scoring layer** merges the two into one threat score with a severity level.
- Alerts are saved to PostgreSQL and pushed in real time to a dashboard through Redis pub/sub and WebSockets.

The threat categories are informed by the Fortinet NSE threat taxonomy.

---

## Demo

> Dashboard screenshot / demo GIF coming soon

<!-- TODO: add docs/demo.gif and docs/dashboard.png after Phase 4 -->

---

## Architecture

```
        Network Flows (CIC-IDS2017 replay)
                     |
                     v
            Traffic Replayer / Ingest
                     |
                     v
          Feature Extraction + Scaling
                     |
          +----------+-----------+
          |                      |
          v                      v
   ML Classifier            Rule Layer
 (probability per         (port scan, SYN flood,
  attack class)            brute force, ...)
          |                      |
          +----------+-----------+
                     |
                     v
           Threat Score + Severity
                     |
                     v
       Redis pub/sub  ---->  PostgreSQL
                     |
                     v
         FastAPI WebSocket endpoint
                     |
                     v
             React Dashboard
```

### Services (Docker Compose)

| Service | Role |
|---|---|
| `api` | FastAPI app: JWT auth, REST endpoints, WebSocket alert stream |
| `engine` | Detection worker: runs the ML model and rules, publishes alerts to Redis |
| `db` | PostgreSQL: users, alerts, flow summaries |
| `redis` | Pub/sub channel between the engine and the API |
| `frontend` | React dashboard |

---

## Results

<!-- TODO: replace with your real numbers from Phase 1. Do not publish placeholder values. -->

| Model | Accuracy | Macro Precision | Macro Recall | Macro F1 |
|---|---|---|---|---|
| Random Forest | TBD | TBD | TBD | TBD |
| XGBoost | TBD | TBD | TBD | TBD |
| Neural Network (TensorFlow) | TBD | TBD | TBD | TBD |

**Per-class recall (selected model):**

| Class | Precision | Recall | F1 |
|---|---|---|---|
| BENIGN | TBD | TBD | TBD |
| DoS | TBD | TBD | TBD |
| PortScan | TBD | TBD | TBD |
| BruteForce | TBD | TBD | TBD |
| WebAttack | TBD | TBD | TBD |

**Evaluation setup:** describe your train/test split here (for example, train on some capture days and test on others) and why it was chosen. Random row-level splits can leak near-duplicate flows between train and test and inflate the numbers.

---

## Features

- **Hybrid detection:** ML classifier plus signature rules, each contributing to one score
- **Explainable alerts:** every alert records which rule fired or which features drove the ML prediction
- **Live alert streaming:** Redis pub/sub to WebSocket to React, no page refresh needed
- **JWT authentication** on all API routes
- **Alert management:** filter by severity and time range, acknowledge alerts
- **Persistent history** in PostgreSQL
- **One-command startup** with Docker Compose
- **CI pipeline** on GitHub Actions (lint, tests, Docker build)

---

## Threat Scoring

Each flow gets a rule score and an ML probability, both between 0 and 1. The final score is:

```python
# Example formula; keep this in sync with the implementation
final_score = max(rule_score, 0.7 * ml_probability + 0.3 * rule_score)
```

**Why this formula:** a strong rule match should never be diluted by a low ML probability (hence the `max`), while the weighted blend lets a confident ML prediction raise the score when no rule fires.

| Final Score | Severity |
|---|---|
| 0.00 to 0.39 | Low |
| 0.40 to 0.69 | Medium |
| 0.70 to 0.89 | High |
| 0.90 to 1.00 | Critical |

<!-- TODO: adjust the thresholds to match your code -->

---

## Signature Rules

| Rule | Trigger |
|---|---|
| Port scan | One source contacts many distinct destination ports within N seconds |
| SYN flood | High SYN rate from a source with few completed handshakes |
| Brute force | Repeated connections to SSH (22) or RDP (3389) from one source |

<!-- TODO: list the rules you actually implemented, with their thresholds -->

---

## Tech Stack

| Category | Technology |
|---|---|
| Frontend | React, Recharts |
| Backend | FastAPI, SQLAlchemy, Alembic |
| Auth | JWT |
| Database | PostgreSQL |
| Messaging | Redis pub/sub, WebSockets |
| Machine Learning | scikit-learn, XGBoost, TensorFlow / Keras |
| Data Processing | Pandas, NumPy |
| DevOps | Docker, Docker Compose, GitHub Actions |
| Testing | pytest |

---

## Dataset

**CIC-IDS2017** from the Canadian Institute for Cybersecurity.

- Labeled network flows captured over five days, with benign traffic and common attacks
- Roughly 80 flow-based features per record (packet counts, flag counts, inter-arrival times, flow duration, and so on)
- Raw labels are merged into five classes: `BENIGN`, `DoS`, `PortScan`, `BruteForce`, `WebAttack`
- Very rare classes (for example Heartbleed and Infiltration) are excluded because there are too few samples to train or evaluate on

Download from [https://www.unb.ca/cic/datasets/ids-2017.html](https://www.unb.ca/cic/datasets/ids-2017.html) (use the "MachineLearningCSV" version) and place the CSV files in `ml/data/`. The dataset is not tracked in this repository.

---

## Project Structure

<!-- TODO: adjust to match your real folder layout -->

```
NetSentry/
│
├── ml/
│   ├── data/                  # CIC-IDS2017 CSVs (not tracked)
│   ├── notebooks/             # Exploration and evaluation notebooks
│   ├── src/
│   │   ├── preprocessing.py   # Cleaning, label merging, splitting, scaling
│   │   ├── train.py           # Model training
│   │   └── evaluate.py        # Metrics and confusion matrix
│   └── models/                # Saved model, scaler, feature list, label encoder
│
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI entry point
│   │   ├── auth.py            # JWT login and route protection
│   │   ├── models.py          # SQLAlchemy models
│   │   ├── routes/            # REST endpoints and WebSocket
│   │   └── engine/
│   │       ├── rules.py       # Signature rules
│   │       ├── scoring.py     # Combined threat score
│   │       ├── detector.py    # ML inference + rule evaluation
│   │       └── replayer.py    # Simulated live traffic stream
│   ├── alembic/               # Database migrations
│   └── tests/
│
├── frontend/
│   └── src/                   # React dashboard
│
├── .github/workflows/ci.yml   # Lint, test, Docker build
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Setup

### Prerequisites

- Docker and Docker Compose
- Python 3.11+ (only needed to retrain the model)

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/NetSentry.git
cd NetSentry
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and set a strong `JWT_SECRET` and database password.

### 3. Download the dataset and train the model

```bash
cd ml
pip install -r requirements.txt
python src/train.py
cd ..
```

This trains the classifiers, prints the evaluation report, and saves the model, scaler, and feature list into `ml/models/`.

### 4. Start the platform

```bash
docker compose up --build
```

### 5. Open the dashboard

Go to `http://localhost:3000` and log in with the credentials configured in your `.env`.

### 6. Replay traffic

```bash
docker compose exec engine python -m app.engine.replayer
```

The replayer feeds dataset flows into the engine as a simulated live stream. Alerts should appear on the dashboard within a few seconds.

---

## API Overview

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login` | Get a JWT access token |
| GET | `/alerts` | List alerts (filter by `severity`, `from`, `to`; paginated) |
| POST | `/alerts/{id}/ack` | Acknowledge an alert |
| WS | `/ws/alerts` | Live alert stream |

Interactive API docs are available at `http://localhost:8000/docs` once the stack is running.

<!-- TODO: confirm these routes match your implementation -->

---

## How It Works

### Training Phase
CIC-IDS2017 is loaded, cleaned (infinite and missing values removed, duplicates dropped, constant columns removed), and its labels are merged into five classes. The data is split, the scaler is fit on the training set only, and the classifiers are trained and compared. The chosen model, scaler, feature column order, and label encoder are saved to `ml/models/`.

### Detection Phase
The replayer streams flows to the engine. For each flow, the engine scales the features, runs the ML classifier, and evaluates the signature rules. The scoring layer combines both results into a threat score and severity, and records the reason (the rule that fired or the top contributing features).

### Alerting Phase
Flows above the alert threshold are stored in PostgreSQL and published to a Redis channel. The API's WebSocket endpoint forwards them to connected dashboard clients, where analysts can filter, inspect, and acknowledge them.

---

## Testing and CI

```bash
cd backend
pytest
```

GitHub Actions runs linting, the pytest suite, and a Docker image build on every push.

---

## Project Status

- [ ] Phase 1: ML core (preprocessing, training, evaluation)
- [ ] Phase 2: Rule layer, scoring, traffic replayer
- [ ] Phase 3: FastAPI backend, database, Redis, WebSocket
- [ ] Phase 4: React dashboard
- [ ] Phase 5: Docker Compose and CI

<!-- TODO: tick these off as you finish each phase -->

---

## Limitations

- Trained on a single benchmark dataset; performance on real production traffic is not guaranteed
- Uses replayed flow records rather than live packet capture
- Signature rules use fixed thresholds that may need tuning per network

---

## Future Work

- Live packet capture and flow extraction
- Automated response actions (for example IP blocking)
- Model retraining pipeline with drift monitoring
- Email or webhook notifications for critical alerts

---

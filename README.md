# IoT Network Anomaly Detection Using Machine Learning

An end-to-end machine learning framework for detecting malicious traffic in IoT networks, built on the **UNSW-NB15** dataset and deployed as a real-time **FastAPI** service.

> **Author:** Shoaib Hossain (ID: 221-15-5949)
> **Department:** Computer Science and Engineering, Daffodil International University, Dhaka, Bangladesh
> **Date:** April 14, 2025

---

## Overview

With billions of IoT devices coming online, the attack surface for cyber threats keeps growing. Traditional signature- or rule-based intrusion detection systems (IDS) struggle with new and sophisticated attacks. This project takes a machine-learning approach: it learns what normal network traffic looks like and flags deviations, covering both known and unknown attacks.

The project covers the full pipeline: data preprocessing, model training and comparison, evaluation, and real-time deployment as a REST API.

## Key Results

Three supervised classifiers were trained under identical conditions and evaluated on the UNSW-NB15 test set (82,332 samples):

| Classifier | Accuracy | Precision | Recall | F1 | FPR | Rank |
|---|---|---|---|---|---|---|
| **Random Forest** | **87.06%** | 0.89 | 0.87 | 0.87 | 0.11 | 1st |
| SVM | 82.00% | 0.86 | 0.82 | 0.80 | 0.18 | 2nd |
| Logistic Regression | 81.00% | 0.84 | 0.81 | 0.80 | 0.19 | 3rd |

**Random Forest** was selected as the best model:

- Attack-class recall of **0.99**, so it misses very few attacks
- ROC AUC of about **0.93** (SVM about 0.88, Logistic Regression about 0.86)
- Normal-class recall is lower (0.73) because of class imbalance (see [Limitations](#limitations))

### Random Forest Per-Class Report

| Class | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Normal (0) | 0.98 | 0.73 | 0.84 | 37,000 |
| Attack (1) | 0.82 | 0.99 | 0.89 | 45,332 |
| **Accuracy** | | | 0.87 | 82,332 |
| Macro Avg | 0.90 | 0.86 | 0.86 | 82,332 |
| Weighted Avg | 0.89 | 0.87 | 0.87 | 82,332 |

## System Architecture

```
UNSW-NB15 CSV
      │
      ▼
Drop id / attack_cat
      │
      ▼
One-Hot Encoding (proto, service, state)
      │
      ▼
Feature Alignment (fill_value=0)
      │
      ▼
StandardScaler
      │
      ├──────────────┬──────────────┐
      ▼              ▼              ▼
Random Forest       SVM     Logistic Regression
      └──────────────┼──────────────┘
                     ▼
        Evaluation & Model Selection
                     │
                     ▼
     joblib Pipeline  →  FastAPI Server  →  Real-time Prediction
```

## Dataset

The project uses the **UNSW-NB15** dataset (Moustafa & Slay, 2015), which contains realistic normal and malicious network traffic.

| Property | Training Set | Testing Set |
|---|---|---|
| Total Records | 175,341 | 82,332 |
| Normal Traffic (0) | 56,000 | 37,000 |
| Attack Traffic (1) | 119,341 | 45,332 |
| Class Ratio (N:A) | 32% : 68% | 45% : 55% |
| Original Features | 45 | 45 |
| After Dropping `id` / `attack_cat` | 43 | 43 |
| After One-Hot Encoding | 194 | 194 |
| Missing Values | 0 | 0 |

The full dataset covers nine attack categories: Fuzzers, Analysis, Backdoors, DoS, Exploits, Generic, Reconnaissance, Shellcode, and Worms.

## Methodology

1. **Load** the training and testing CSV files.
2. **Drop** redundant columns (`id`, `attack_cat`).
3. **Split** features `X` and the binary label `y` (0 = Normal, 1 = Attack).
4. **One-hot encode** the categorical features `proto`, `service`, and `state` with `pd.get_dummies()`, then align train/test columns with `fill_value=0`.
5. **Standardize** features with `StandardScaler`, fitted on the training set only to prevent data leakage.
6. **Train** Random Forest (100 trees), SVM, and Logistic Regression (`max_iter=1000`).
7. **Evaluate** with accuracy, precision, recall, F1-score, confusion matrix, and ROC/AUC.
8. **Select** the model with the best weighted F1-score.
9. **Serialize** the full pipeline (encoding + scaling + model) with `joblib`.
10. **Serve** the pipeline through a FastAPI `/predict` endpoint.

## API Deployment

The best model is served as a RESTful API with FastAPI and Uvicorn.

| Property | Value |
|---|---|
| Model file | `iot_attack_detection_pipeline.joblib` |
| Framework | FastAPI 0.135.2 + Uvicorn 0.42.0 |
| Host | `0.0.0.0:8000` |
| `GET /` | HTTP 200 OK |
| `POST /predict` | HTTP 200 OK |
| Response fields | `prediction`, `confidence` (%), `input_received` |

**Request flow:** Client → `POST /predict` → Pydantic validation → sklearn pipeline → Random Forest prediction → JSON response.

**Sample response:**

```json
{
  "prediction": "Attack",
  "confidence": 92.4
}
```

The API can be exposed publicly for testing with a tunnel such as `localtunnel`.

## Tech Stack

Experiments were run on Google Colab (Python 3.12).

| Component | Version |
|---|---|
| Python | 3.12 |
| pandas | 2.2.2 |
| numpy | 2.0.2 |
| scikit-learn | 1.6.1 |
| matplotlib | 3.10.0 |
| seaborn | 0.13.2 |
| joblib | 1.5.3 |
| FastAPI | 0.135.2 |
| uvicorn | 0.42.0 |
| pyngrok / localtunnel | 8.0.0 |

## Getting Started

> Adjust file names and paths to match this repository's layout.

```bash
# 1. Clone the repository
git clone https://github.com/shoaibhossain5949/IoT-Network-Anomaly-Detection-ML.git
cd IoT-Network-Anomaly-Detection-ML

# 2. Install dependencies
pip install pandas numpy scikit-learn matplotlib seaborn joblib fastapi uvicorn

# 3. Place the UNSW-NB15 training and testing CSV files in the project directory

# 4. Run the notebook / training script to build the pipeline
#    (produces iot_attack_detection_pipeline.joblib)

# 5. Start the API server
uvicorn main:app --host 0.0.0.0 --port 8000
```

Then open `http://localhost:8000/docs` for the interactive Swagger UI and try the `/predict` endpoint.

## Limitations

- Trained for **binary** classification (normal vs. attack), not multi-class attack identification.
- The training data is imbalanced (68% attack), which lowers normal-class recall (0.73) and can affect rare attack types.
- Built and tested on a benchmark dataset in a lab environment, not on live IoT traffic, so real-world performance and scalability are unverified.
- Some false positives and false negatives remain.

## Future Work

- **Deep learning:** LSTM, CNN-BiLSTM, and Transformer models for temporal traffic patterns.
- **Imbalance handling:** SMOTE, Borderline-SMOTE, and cost-sensitive learning to raise normal-class recall.
- **Multi-class detection:** classify all nine UNSW-NB15 attack categories.
- **Boosting:** XGBoost and LightGBM.
- **Explainability:** SHAP for interpretable predictions.

## Keywords

IoT Security · Anomaly Detection · Machine Learning · Random Forest · FastAPI · Intrusion Detection System · Network Security

## References

1. N. Moustafa and J. Slay, "UNSW-NB15: a comprehensive data set for network intrusion detection systems," *MilCIS*, 2015.
2. S. More et al., "Enhanced intrusion detection systems performance with UNSW-NB15 data analysis," *Algorithms*, vol. 17, no. 2, 2024.
3. S. M. Kasongo and Y. Sun, "Performance analysis of intrusion detection systems using a feature selection method on the UNSW-NB15 dataset," *Journal of Big Data*, vol. 7, 2020.
4. R. A. Sharma et al., "Lumen: a framework for developing and evaluating ML-based IoT network anomaly detection," *CoNEXT '22*, 2022.
5. Y. Mirsky et al., "Kitsune: An ensemble of autoencoders for online network intrusion detection," 2018.
6. L. Breiman, "Random forests," *Machine Learning*, vol. 45, 2001.

The full reference list is available in the project report.

## Acknowledgements

Thanks to the supervisor and faculty members for their guidance and support throughout this work.

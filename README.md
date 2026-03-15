# Network Intrusion Detection System — Two-Layer IDS with ML & Deep Learning

Detecting known and zero-day network attacks using a two-layer architecture:
a supervised XGBoost multiclass classifier combined with an unsupervised 
autoencoder anomaly detector, built on the NSL-KDD benchmark dataset.

---

## The Problem

Traditional intrusion detection systems fail in two ways:
- They misclassify rare attack types (R2L, U2R) due to limited training data
- They cannot detect zero-day attacks — threats never seen during training

This project addresses both with a two-layer pipeline.

---

## Architecture
```
Incoming connection
        │
        ├──► Layer 1: Autoencoder
        │    Trained on normal traffic only
        │    Flags anything it struggles to reconstruct → anomaly score
        │
        └──► Layer 2: XGBoost Classifier
             Classifies known attack families: DoS / Probe / R2L / U2R
             Provides confidence score per prediction
             
Combined alert levels: CLEAR / SUSPICIOUS / ALERT / HIGH ALERT
```

---

## Dataset

**NSL-KDD** — 125,973 training records, 22,544 test records, 41 features.  
Attack families: DoS, Probe, R2L, U2R + unseen zero-day types in test set.  
Source: Canadian Institute for Cybersecurity

---

## Notebooks

| Notebook | Description |
|---|---|
| `01_eda.ipynb` | Class distribution, attack breakdown, feature types |
| `02_preprocessing.ipynb` | Label encoding, standard scaling, SMOTE balancing |
| `03_models.ipynb` | Logistic Regression, Random Forest, XGBoost — multiclass comparison |
| `03_r2l.ipynb` | R2L recall analysis — class weights, threshold tuning, dedicated classifier |
| `04_shap_explanations.ipynb` | Global + per-attack-family SHAP feature importance |
| `05_autoencoder.ipynb` | Autoencoder anomaly detection — training, threshold optimisation, zero-day evaluation |
| `06_realtime_pipeline.ipynb` | Live stream simulation — two-layer pipeline with SOC analyst output |

---

## Results

### Supervised Layer — XGBoost Multiclass

| Class | Precision | Recall | F1 |
|---|---|---|---|
| DoS | 0.99 | 1.00 | 0.99 |
| Probe | 0.83 | 1.00 | 0.91 |
| R2L | 0.99 | 0.17 | 0.29 |
| U2R | 0.53 | 0.27 | 0.36 |
| normal | 0.84 | 0.97 | 0.90 |
| **Macro F1** | | | **0.69** |

> R2L recall (0.17 baseline) was investigated in `03_r2l.ipynb`.
> Threshold tuning at 0.05 improved R2L recall to 0.30 while raising overall
> macro F1 to 0.73 with precision held at 0.985.

### Unsupervised Layer — Autoencoder (v2)

| Metric | Score |
|---|---|
| ROC-AUC | 0.9437 |
| Macro F1 | 0.87 |
| Zero-day detection | **93.8%** |
| R2L detection | 62.6% |

### Two-Layer Pipeline (live simulation)

| Metric | Result |
|---|---|
| Overall detection rate | 88.8% |
| Zero-day detection | 97.5% |
| Connections flagged SUSPICIOUS (autoencoder only) | 26 — missed by XGBoost alone |

---

## Key Findings

**1. Zero-day blind spot is real and measurable**  
XGBoost scores 0% on unseen attack types. The autoencoder catches 93.8% of 
the same connections with no labels — purely from learning normal traffic patterns.

**2. R2L is the hardest class across all models**  
Root cause: train/test distribution shift. R2L in KDDTest+ contains attack 
subtypes not present in KDDTrain+. Threshold tuning doubled recall vs baseline.

**3. The two layers are complementary, not redundant**  
In the live simulation, 26 connections were flagged SUSPICIOUS by the 
autoencoder that XGBoost labelled as normal — demonstrating genuine 
added value from the unsupervised layer.

**4. Threshold is configurable by operational context**  
80th percentile threshold maximises detection sensitivity (97.5% zero-day catch 
rate, 27.5% false alarm rate). For lower false alarms, raise to 95th percentile 
(8% false alarm rate, 76.4% zero-day detection). Tradeoff is explicit and tunable.

---

## SHAP Explainability

Per-attack-family SHAP analysis reveals distinct feature signatures:
- **DoS**: dominated by `serror_rate`, `dst_host_serror_rate`
- **Probe**: driven by `dst_host_srv_count`, `dst_host_same_src_port_rate`  
- **R2L**: `logged_in`, `num_compromised` are top signals
- **U2R**: `root_shell`, `num_root` distinguish privilege escalation

---

## Tech Stack

Python · scikit-learn · XGBoost · TensorFlow/Keras · SHAP · 
imbalanced-learn · pandas · numpy · matplotlib · seaborn

---

## Setup
```bash
git clone https://github.com/07healer/network-intrusion-detection
cd network-intrusion-detection
pip install -r requirements.txt
```

Download NSL-KDD dataset from [Kaggle](https://www.kaggle.com/datasets/hassan06/nslkdd)  
and place `KDDTrain+.txt` and `KDDTest+.txt` in the `data/` folder.

---

## Next Steps

- Extend autoencoder to variational architecture (VAE) for better anomaly scores
- Evaluate on CICIDS2017 for real-world traffic distribution  
- Expose best model as a lightweight REST API
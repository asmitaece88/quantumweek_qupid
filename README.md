# Quantum Fraud Detection with Hybrid Variational Circuits

This repository provides a proof-of-concept (PoC) implementation of a **hybrid quantum-classical fraud detection engine**. The project evaluates **Variational Quantum Classifiers (VQCs)**—built with **PennyLane** and **Qiskit**—against a feature-matched **XGBoost** classical benchmark to identify rare, high-risk payment fraud transactions in financial telemetry data.

---

## 1. Executive Summary

Financial payment fraud is a classic **rare-event detection problem**. In real-world payment networks, legitimate transactions drastically outnumber fraudulent attempts.

* **The Accuracy Paradox**: Evaluating models using standard accuracy is misleading. A trivial classifier predicting "legitimate" for every transaction achieves ~98% accuracy while failing to detect a single fraudulent event.
* **Core Objective**: Optimize for fraud ranking quality using **Precision-Recall Area Under Curve (PR-AUC)**, **Recall**, **Precision**, and **False Positive alert counts** to route the highest-risk transactions into a manageable review queue for human investigators.

---

## 2. Dataset Architecture & Column Analysis

The benchmark relies on `fraud_detection_dataset.csv`, a synthetic financial transaction dataset containing **2,000 records** with a severe **2.0% fraud class imbalance** (1,960 legitimate transactions vs. 40 fraudulent cases).

### Detailed Column Breakdown

| Column Name | Data Type | Range / Values | Mean (Overall) | Fraud vs. Non-Fraud Behavior | Role & Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `transaction_amount` | Continuous Float | \$2.30 – \$688.44 | \$56.28 | **Fraud**: \$63.35 mean<br>**Legit**: \$56.14 mean | Monetary value of the payment attempt in USD. Fraudulent transactions exhibit higher variance and slightly elevated average transaction values. |
| `device_trust_score` | Continuous Float | 0.082 – 0.996 | 0.711 | **Fraud**: 0.638 mean<br>**Legit**: 0.712 mean | Normalized trust metric calculated from device fingerprinting, IP reputation, and hardware signals. Lower scores reflect compromised or unrecognized devices. |
| `merchant_risk_score` | Continuous Float | 0.0003 – 0.820 | 0.240 | **Fraud**: 0.075 mean<br>**Legit**: 0.244 mean | Estimated merchant risk signal used directly by the benchmark. Fraud cases in this synthetic dataset cluster at lower values. |
| `location_distance_km` | Continuous Float | 0.35 km – 96.32 km | 19.57 km | **Fraud**: 19.14 km mean<br>**Legit**: 19.58 km mean | Geographic distance between the transaction initiation point and the cardholder's historical billing centroid. |
| `transactions_last_hour` | Integer | 0 – 7 counts | 1.08 counts | **Fraud**: 1.60 mean<br>**Legit**: 1.07 mean | Velocity metric measuring total transaction attempts from the same card/account within the preceding 60 minutes. |
| `fraud` | Binary Indicator | 0 or 1 | 0.020 (2.0%) | **0**: 1,960 (98.0%)<br>**1**: 40 (2.0%) | **Target Variable**. $0 = \text{Legitimate Transaction}$, $1 = \text{Fraudulent Transaction}$. |

### Feature Selection

The 2-qubit benchmark uses two dataset columns directly:

| Feature | Values near 0 | Values near 1 |
| :--- | :--- | :--- |
| `device_trust_score` | Low device trust: an unfamiliar, less-established or potentially suspicious device. | High device trust: a familiar, well-established device. |
| `merchant_risk_score` | Low estimated merchant risk. | High estimated merchant risk: a merchant category or behaviour assessed as more risky. |

$$\mathbf{x} = \left[ \text{device trust score},\; \text{merchant risk score} \right]$$

---

## 3. Data Preprocessing & Split Pipeline

To ensure a rigorous and fair benchmark between classical and quantum models, both paradigms execute on the exact same data splits and evaluation contracts.

* **Held-Out Test Set**: 600 transactions (**12 actual fraud cases, 2.0% fraud rate**) reserved exclusively for final evaluation ($Seed = 31$).
* **Training Subset Oversampling**: The training fitting set (1,400 transactions) is oversampled for the VQC fitting step to produce a balanced 240-sample set (**47.5% fraud rate**) to prevent gradient starvation in variational parameter updates.
* **Feature Angle Scaling**: Both trust features are rescaled from $[0, 1]$ to the interval $[-\pi, \pi]$:
  $$x_i \mapsto x_i' = 2\pi x_i - \pi$$

---

## 4. Model Approaches: Classical vs. Hybrid VQC

```
                    ┌─────────────────────────────────────────────────────────┐
                    │               Raw Payment Telemetry                     │
                    │      (device_trust_score, merchant_risk_score)          │
                    └────────────────────────────┬────────────────────────────┘
                                                 │
                        ┌────────────────────────┴────────────────────────┐
                        ▼                                                 ▼
        ┌───────────────────────────────┐                 ┌───────────────────────────────┐
        │     Classical Pipeline        │                 │    Hybrid Quantum Pipeline    │
        │           (XGBoost)           │                 │    (PennyLane / Qiskit VQC)   │
        └───────────────┬───────────────┘                 └───────────────┬───────────────┘
                        │                                                 │
                        │                                  Angle Encoding: x -> [-π, π]
                        │                                                 │
                        │                                 2-Qubit Variational Ansatz
                        │                                 (RY/RZ + CNOT Entanglers)
                        │                                                 │
                        │                                 Pauli-Z Expectation Measurement
                        │                                                 │
                        │                                 Classical COBYLA Optimizer
                        │                                                 │
                        ▼                                                 ▼
        ┌───────────────────────────────┐                 ┌───────────────────────────────┐
        │  Probability Score [0.0, 1.0] │                 │  Probability Score [0.0, 1.0] │
        └───────────────┬───────────────┘                 └───────────────┬───────────────┘
                        │                                                 │
                        └────────────────────────┬────────────────────────┘
                                                 │
                                                 ▼
                                ┌──────────────────────────────────┐
                                │   PR-AUC & Fraud Review Queue    │
                                └──────────────────────────────────┘
```

### Classical Baseline: XGBoost

* **Architecture**: Feature-matched Gradient Boosted Decision Trees trained on the 2 selected trust features.
* **Hyperparameters**: `max_depth=3`, `n_estimators=100`, `learning_rate=0.05`, `scale_pos_weight=49` to account for class imbalance.
* **Mechanism**: Partitions feature space using orthogonal decision boundaries to compute fraud probability logits.

### Quantum Approach: Hybrid Variational Quantum Classifier (VQC)

* **Frameworks**: Implemented via **PennyLane** (`default.qubit`) and **Qiskit Aer** (`StatevectorSampler`).
* **Qubit Topology**: 2-qubit quantum register corresponding to the 2 input features.
* **Feature Encoding**: Angle embedding where feature values $x_1', x_2' \in [-\pi, \pi]$ drive single-qubit rotation gates:
  $$|\psi_0\rangle = U_{enc}(\mathbf{x}')|00\rangle = \left( R_Y(x_1') \otimes R_Y(x_2') \right) |00\rangle$$
* **Variational Ansatz**: Parameterized entangling circuit with $L$ layers:
  $$U_{var}(\boldsymbol{\theta}) = \prod_{l=1}^L \left[ \text{CNOT}_{0,1} \cdot \left( R_Z(\theta_{l,1}) \otimes R_Z(\theta_{l,2}) \right) \cdot \left( R_Y(\theta_{l,3}) \otimes R_Y(\theta_{l,4}) \right) \right]$$
* **Measurement & Post-Processing**: Expectation value of Pauli-$Z$ operator on the primary qubit is mapped to a calibrated fraud probability $P(\text{Fraud}) \in [0, 1]$:
  $$\langle Z_0 \rangle = \langle \psi(\mathbf{x}, \boldsymbol{\theta}) | Z_0 | \psi(\mathbf{x}, \boldsymbol{\theta}) \rangle \implies P(\text{Fraud}) = \frac{1 - \langle Z_0 \rangle}{2}$$
* **Optimization Loop**: Classical **COBYLA** (Constrained Optimization BY Linear Approximation) optimizer updates circuit parameters $\boldsymbol{\theta}$ across 100 iterations.

---

## 5. Experimental Results & Performance Comparison

Evaluated on the **untouched 600-transaction test set** (12 fraud cases):

| Metric | PennyLane VQC (Simulation PoC) | XGBoost (Classical Baseline) | Operational Significance |
| :--- | :--- | :--- | :--- |
| **PR-AUC** | **0.3979** | **0.4103** | VQC closely tracks XGBoost in precision-recall ranking power. |
| **Fraud Recall** | **100.0% (12 / 12)** | **91.7% (11 / 12)** | PennyLane VQC captured **every single fraudulent transaction**. |
| **Precision** | **5.9%** | **28.9%** | XGBoost maintains higher precision in simulation. |
| **False Positives** | **193** | **27** | VQC simulation flagged more false alarms; threshold tuning required. |
| **Held-Out Test Size**| **600 transactions** | **600 transactions** | Identical evaluation split ($Seed = 31$). |

---

## 6. Phase 2 Roadmap: IBM Quantum Cloud & Hardware Error Mitigation

While Phase 1 validated the hybrid workflow on statevector simulators, **Phase 2 moves execution to real quantum hardware on the IBM Quantum Cloud**.

```
  Phase 1: Local Simulator PoC               Phase 2: IBM Quantum Cloud Prototype
┌───────────────────────────────┐          ┌─────────────────────────────────────────┐
│ • PennyLane / Qiskit Aer      │          │ • IBM Quantum Heron / Eagle QPU         │
│ • Statevector Execution       │          │ • Transpilation & Layout Mapping        │
│ • Ideal / Zero-Noise          │ ───────► │ • Readout Error Mitigation (M3)         │
│ • PR-AUC: 0.3979              │          │ • Zero-Noise Extrapolation (ZNE)        │
│ • Benchmark Comparison        │          │ • Fair comparison with XGBoost          │
└───────────────────────────────┘          └─────────────────────────────────────────┘
```

### Hardware Execution Architecture

1. **Backend-Aware Transpilation**: Mapping 2-qubit virtual circuits to native QPU coupling maps, optimizing single-qubit gate decompositions ($R_z, \sqrt{X}, X$) and minimizing CNOT counts.
2. **Readout Error Mitigation (M3)**: Matrix Inversion and Measurement Mitigation (`qiskit-m3`) to correct assignment errors on noisy physical measurements.
3. **Zero-Noise Extrapolation (ZNE)**: Intentionally scaling circuit noise to extrapolate zero-noise expectation values $\langle Z_0 \rangle_{noise \to 0}$.
4. **Performance Validation**: Simulation VQC results closely track XGBoost. Real IBM Quantum hardware runs with error mitigation will test whether the VQC can maintain or improve its ranking quality under realistic noise; outperforming XGBoost is a hypothesis to validate, not a guaranteed outcome.

---

## 7. Software Dependencies & Installation

Required dependencies specified in `requirements.txt`:

```text
qiskit>=1.0.0
qiskit-aer
qiskit_machine_learning
qiskit-algorithms
pennylane>=0.35.0
numpy
pandas
matplotlib
scikit-learn
scipy
xgboost
jupyter
pylatexenc
```

### Quickstart

```bash
# Clone repository
git clone https://github.com/your-org/quantum-fraud-detection.git
cd quantum-fraud-detection

# Install dependencies
pip install -r requirements.txt

# Run prototype notebook
jupyter notebook quantum_vqc_fraud_detection.ipynb
```

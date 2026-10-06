================================================================================
QUANTUM FRAUD DETECTION WITH HYBRID VARIATIONAL CIRCUITS (VQC)
================================================================================
Project: Q-Hack India 2026 Prototype
Track: Hybrid Quantum-Classical Machine Learning for Rare Event Detection
Team: Team Qupid

--------------------------------------------------------------------------------
1. EXECUTIVE OVERVIEW
--------------------------------------------------------------------------------
This repository implements a Round 1 prototype for payment fraud detection using a
hybrid quantum-classical Variational Quantum Classifier (VQC) compared against
a feature-matched XGBoost classical benchmark. 

Because financial fraud is a rare-event detection problem, standard accuracy
is a misleading metric (predicting 'legitimate' for all transactions yields ~98%
accuracy while catching zero fraud). Therefore, this project evaluates fraud
ranking capability using Precision-Recall Area Under Curve (PR-AUC), Recall,
Precision, and False-Positive counts to optimize alert queues for human fraud
investigators.

--------------------------------------------------------------------------------
2. DATASET CHARACTERISTICS & EXTREME CLASS IMBALANCE
--------------------------------------------------------------------------------
The synthetic financial dataset (`fraud_detection_dataset.csv`) contains:
  * Total Transactions: 2,000
  * Legitimate Cases: 1,960 (98.0%)
  * Fraudulent Cases: 40 (2.0% extreme fraud rate)

Selected Feature Pair for Fair Comparison:
  Both quantum and classical models are trained and evaluated on the exact same
  two input features:
  1. `device_trust_score`: Trustworthiness metric derived from device telemetry.
  2. `merchant_trust_score`: Merchant reliability metric.

Feature Mislabeling Correction:
  The dataset column originally named `merchant_risk_score` was corrected to
  `merchant_trust_score`. Analysis confirmed that higher values correspond to
  higher merchant trustworthiness (lower risk), while lower values indicate
  suspicious merchants. No numerical feature values or fraud labels were changed.

Data Split & Balance Strategy:
  * Evaluation Contract: An untouched held-out test set of 600 transactions
    containing 12 fraud cases (2.0% fraud rate) was reserved (seed 31).
  * VQC Fit Subset Oversampling: To train the variational quantum circuit on
    an imbalanced domain, fraud cases were oversampled exclusively in the VQC
    training fitting subset (240 samples, 47.5% fraud). The held-out test set
    remained entirely untouched.

--------------------------------------------------------------------------------
3. TECHNICAL APPROACH & ARCHITECTURE
--------------------------------------------------------------------------------
A. Classical Benchmark (XGBoost):
   * Architecture: Gradient boosted decision trees using the feature-matched pair
     (`device_trust_score`, `merchant_trust_score`).
   * Purpose: Serves as the state-of-the-art classical baseline evaluated on the
     untouched held-out test set.

B. Quantum Approach (Hybrid Variational Quantum Classifier - VQC):
   * Frameworks: Developed using PennyLane, Qiskit, Qiskit Aer, and Qiskit Machine
     Learning.
   * Feature Map (Angle Encoding): Both continuous features are scaled to [-pi, pi]
     and embedded into a 2-qubit Hilbert space via single-qubit rotations.
   * Variational Ansatz: Parameterized entangling layers consisting of single-qubit
     rotation gates (RY, RZ) and CNOT entanglers to capture non-linear feature
     interactions in compact quantum state space.
   * Hybrid Optimization Loop: Classical optimizer (COBYLA) updates trainable
     circuit parameters iteratively.
   * Fraud Probability Output: Pauli-Z expectation value measurement is mapped
     directly to transaction fraud probability.

--------------------------------------------------------------------------------
4. POC SIMULATION RESULTS (HELD-OUT TEST SET: 600 TRANSACTIONS, 12 FRAUD CASES)
--------------------------------------------------------------------------------
+--------------------+---------+--------+-----------+-----------------+
| Model              | PR-AUC  | Recall | Precision | False Positives |
+--------------------+---------+--------+-----------+-----------------+
| PennyLane VQC sim  | 0.3979  | 100.0% |   5.9%    |      193        |
| XGBoost Benchmark  | 0.4103  |  91.7% |  28.9%    |       27        |
+--------------------+---------+--------+-----------+-----------------+

Key Findings:
  * PennyLane VQC achieved a PR-AUC of 0.3979, closely matching classical XGBoost
    (0.4103) while capturing 100% of genuine fraud cases (12/12) in simulation.
  * XGBoost provided higher precision (28.9%) and fewer false positives (27).

--------------------------------------------------------------------------------
5. PHASE 2 IMPLEMENTATION ROADMAP: IBM QUANTUM CLOUD HARDWARE
--------------------------------------------------------------------------------
Phase 1 (Current): Local Simulator PoC
  * Executed on local statevector simulators (PennyLane / Qiskit Aer).
  * Established baseline feature encoding and optimization pipelines.

Phase 2 (Upcoming): IBM Quantum Cloud Prediction Prototype
  * Transpilation: Backend-aware transpilation matching native physical qubit
    coupling maps on IBM Quantum processors.
  * Hardware Execution: Running variational circuits on real quantum hardware.
  * Error Mitigation: Implementing readout error mitigation (M3) and Zero-Noise
    Extrapolation (ZNE) to mitigate gate noise and decoherence.
  * Hypothesis: With proper error mitigation and expanded expressive ansätze,
    real quantum hardware is projected to surpass classical XGBoost ranking
    performance by exploring higher-dimensional entangling spaces.

--------------------------------------------------------------------------------
6. REPOSITORY STRUCTURE & DEPENDENCIES
--------------------------------------------------------------------------------
Files:
  * `quantum_vqc_fraud_detection.ipynb`: Main prototype execution notebook.
  * `fraud_detection_dataset.csv`: 2,000 transaction dataset with fraud labels.
  * `requirements.txt`: Python package manifest (qiskit, pennylane, xgboost,
    scikit-learn, pandas, numpy, matplotlib).
  * `README.md`: Project summary documentation.

Dependencies Installation:
  $ pip install -r requirements.txt
================================================================================

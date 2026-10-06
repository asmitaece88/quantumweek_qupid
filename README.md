> **Dataset clarification:** We did not change any dataset values because the data itself was usable. The issue was only that `merchant_risk_score` was mislabeled, so we renamed it to `merchant_trust_score` to match what the existing values actually represent.
>
> **Round 1 Prototype:** A hybrid quantum-classical fraud detection
> experiment comparing Variational Quantum Classifiers implemented with
> Qiskit and PennyLane against an XGBoost baseline on the same
> highly-imbalanced synthetic dataset.

## Round 1 Prototype Update

This repository contains the Round 1 prototype for a hybrid
quantum-classical fraud detection use case.

### Dataset

The synthetic dataset contains:

- 2,000 transactions
- 40 fraudulent transactions
- 2% fraud rate

The current benchmark uses two features:

- `device_trust_score`
- `merchant_trust_score`

The column previously named `merchant_risk_score` was renamed to
`merchant_trust_score`.

The existing values behave as a trust score:

- Higher value = more trusted merchant
- Lower value = less trusted / potentially suspicious merchant

No numerical feature values or fraud labels were modified as part of
this change. Only the feature name and its documentation were corrected.

### Fair Model Comparison

Both the quantum and classical models receive the same two input
features:

```text
device_trust_score
merchant_trust_score
```

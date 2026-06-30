# Fraud Scoring Service (svc-16)

**Domain:** Fraud
**Stack:** Python / FastAPI
**Database:** PostgreSQL (`fraud_scoring_db`)
**Deployment:** AWS EKS

## Component Overview

The Fraud Scoring Service is where signals converge. It aggregates data from three independent fraud detection sources, feeds them into a machine learning model, and produces a unified fraud risk score for each proposal. This score is the primary input to the Fraud Decision Service, which renders the accept/reject/review verdict.

The three input signals are:

1. **Device Fingerprint Signal** -- from Device Fingerprint Service (svc-13): device risk score, number of identities associated with the device, emulator/proxy flags.
2. **Bureau Signal** -- from Fraud Bureau Integration (svc-15): ClearCheck's normalized fraud score, CPF alert flags, address mismatch indicators.
3. **Internal Behavioral Signal** -- computed locally within this service: application velocity (how many proposals this CPF has submitted recently), data consistency checks (does the declared income match the bureau's estimate?), and time-of-day patterns.

### Components

**Signal Collector:** An event-driven component that listens on SQS for `DeviceFingerprinted`, `FraudBureauCheckCompleted`, and `ProposalCreated` events. For each proposal, it accumulates signals in a PostgreSQL staging table until all three are present (or until a configurable timeout expires, at which point it scores with whatever signals are available).

**Feature Engineer:** Transforms raw signals into a feature vector suitable for the ML model. This includes normalization, one-hot encoding of categorical variables, and derivation of composite features (e.g., "bureau_score * device_risk" interaction term).

**Scoring Model:** A scikit-learn GradientBoosting classifier trained on historical fraud outcomes. The model outputs a probability of fraud (0.0 to 1.0), which is scaled to an integer score (0-100). Model artifacts are loaded from S3 at startup, versioned by a `FRAUD_MODEL_VERSION` environment variable.

**Score Publisher:** Persists the score and feature breakdown in PostgreSQL (for model monitoring and explainability) and publishes a `FraudScoreComputed` event to SNS.

## Data Flow

The scoring process is inherently asynchronous and event-driven because the three input signals arrive at different times:

1. `ProposalCreated` event arrives -- a scoring record is initialized.
2. `DeviceFingerprinted` event arrives -- device signal is stored against the scoring record.
3. `FraudBureauCheckCompleted` event arrives -- bureau signal is stored.
4. Once all three signals are present (or after a 60-second timeout), the Feature Engineer builds the feature vector and the Scoring Model computes the fraud probability.
5. The score, feature importances, and individual signal contributions are persisted.
6. `FraudScoreComputed` event is published to SNS.

If a signal is missing at scoring time (e.g., ClearCheck was down), the model handles it via trained imputation -- it has been trained on datasets with missing features and can still produce a calibrated score, albeit with wider confidence intervals.

## Integration Patterns

- **Inbound (async):** SQS queues for three event types from three different domain services.
- **Outbound (async):** `FraudScoreComputed` event to SNS, consumed by `fraud-decision-service`.
- **PostgreSQL:** Stores scoring records, feature vectors, and model outputs for monitoring and retraining.
- **S3:** Reads model artifacts at startup.

## ADR-001: Aggregation-Then-Score vs. Real-Time Streaming Scores

**Context:** The team evaluated two approaches: (1) wait for all signals, then score once; (2) score incrementally as each signal arrives, updating the score in real time. The streaming approach would give the Fraud Decision Service earlier signal but requires a model that handles partial inputs well.

**Decision:** Wait for all signals (with a 60-second timeout) before scoring. This produces a single, high-confidence score rather than multiple provisional scores.

**Consequences:**
- Simpler downstream logic: the Fraud Decision Service receives exactly one score per proposal.
- End-to-end latency for fraud scoring is bounded by the slowest signal source (typically the bureau check at 3-5 seconds).
- The 60-second timeout ensures proposals are not blocked indefinitely if a signal source fails. Missing signals are handled by the model's imputation capability.

## ADR-002: scikit-learn over Deep Learning Frameworks

**Context:** The ML team considered deep learning (PyTorch/TensorFlow) for the fraud model. However, the feature set is tabular (not image/text), the dataset is modest (150K labeled proposals), and interpretability is a regulatory requirement.

**Decision:** Use scikit-learn's GradientBoosting classifier. It achieves an AUC of 0.94 on the test set, provides feature importances natively, and runs inference in under 5ms on CPU.

**Consequences:**
- No GPU required for inference, reducing infrastructure costs.
- Feature importances are stored with every score, enabling explainability reports for compliance.
- If the dataset grows to 1M+ records or if non-tabular features (e.g., document images) are added, migration to XGBoost or a deep learning model may be warranted.

# Document Classifier Service (svc-07)

**Domain:** Documents
**Stack:** Python / FastAPI
**Database:** None (model artifacts loaded from S3 at startup)
**Deployment:** AWS EKS

## Component Overview

When a customer uploads a document to CredityFlow, the system needs to determine what kind of document it is: a CNH (driver's license), RG (national ID), proof of address, payslip, property deed, vehicle registration, or one of a dozen other types. The Document Classifier Service answers this question using a machine learning classification model.

The service exposes a single synchronous REST endpoint and also consumes messages from SQS for batch classification. It does not persist any data -- classification results are returned to the caller or published as events.

**Internal architecture:**

- **Model Runtime** -- loads a pre-trained image classification model (EfficientNet-B0, fine-tuned on CredityFlow's document corpus) into memory at startup. The model weights are pulled from an S3 model registry bucket. Inference runs on CPU; the model is lightweight enough that GPU is unnecessary for the current throughput.
- **Prediction API** -- a `/classify` POST endpoint that accepts an image (as a URL or base64 payload) and returns the predicted document type along with a confidence score. Responses are typically returned in under 200ms.
- **SQS Consumer** -- listens on `document-classification-queue` for `DocumentUploaded` events. Fetches the image from S3, classifies it, and publishes a `DocumentClassified` event to SNS.
- **Fallback Logic** -- when confidence is below 80%, the service returns the top-3 predictions and flags the document for human review via backoffice.

## Data Flow

Two entry points feed into the same classification logic:

**Synchronous path:** The Document Upload Service or backoffice calls the `/classify` endpoint directly when immediate classification is needed (e.g., real-time upload UX where the app confirms "We detected this is your driver's license").

**Asynchronous path:** A `DocumentUploaded` event arrives on the SQS queue. The service downloads the image, classifies it, and emits `DocumentClassified` with the result. This is the primary path during normal onboarding.

Both paths produce identical outputs; the difference is delivery mechanism.

## Integration Patterns

- **REST (sync):** Called by `document-upload-service` and backoffice for real-time classification.
- **SQS (async):** Subscribed to `DocumentUploaded` events for batch processing.
- **SNS (outbound):** Publishes `DocumentClassified` events consumed by `document-validation-service` and `proposal-service`.
- **S3:** Reads document images for classification and loads model artifacts at boot time.

The service is horizontally scalable. Each replica loads the model into memory (~150 MB), so memory requests are set accordingly in the Kubernetes deployment manifest.

## ADR-001: On-Premise ML Inference vs. AWS SageMaker

**Context:** The team evaluated hosting the classification model on AWS SageMaker for managed scaling and monitoring. However, SageMaker endpoints introduce network latency (10-30ms per call), incur per-invocation costs, and require a separate deployment pipeline.

**Decision:** Host the model directly within the FastAPI service, loading weights from S3 at startup. Inference runs in-process using ONNX Runtime for optimized CPU execution.

**Consequences:**
- Lower latency (~50ms inference vs ~80-110ms via SageMaker).
- Simpler deployment: model updates are just S3 uploads followed by a rolling restart.
- The team must handle scaling and health monitoring themselves, but KEDA-based autoscaling on SQS queue depth covers this adequately.
- Model versioning is managed via S3 object versioning and a `MODEL_VERSION` environment variable.

## ADR-002: EfficientNet-B0 over Larger Architectures

**Context:** During model selection, the ML team benchmarked EfficientNet-B0, ResNet-50, and EfficientNet-B4 against the labeled document dataset (22,000 images, 14 classes).

**Decision:** EfficientNet-B0 was selected. It achieved 94.2% accuracy (vs. 95.1% for B4 and 93.8% for ResNet-50) while requiring 5x less memory and 3x less inference time than B4.

**Consequences:**
- Each pod requires only 512 MB of memory for the model, keeping infrastructure costs low.
- The 0.9% accuracy gap vs. B4 is offset by the fallback-to-human-review mechanism for low-confidence predictions.
- If accuracy requirements tighten in the future, upgrading to B4 is straightforward -- same training pipeline, just swap the model artifact.

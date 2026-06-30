# Face Match Service (svc-12)

**Domain:** Biometrics
**Stack:** Python / FastAPI
**Database:** PostgreSQL (`face_match_db`)
**Deployment:** AWS EKS (GPU-enabled node group)

## Component Overview

The Face Match Service performs facial comparison between a customer's selfie and their identity document photo. Its job is straightforward in concept but nuanced in implementation: generate a 512-dimensional face embedding for each image and compute their cosine similarity. If the similarity exceeds a calibrated threshold, the faces are considered a match.

Unlike the Liveness Worker, this service does maintain a database. PostgreSQL stores face embeddings and match results, enabling re-matching against new document photos without requiring the customer to retake their selfie.

### Components

**Face Embedding Generator:** Uses a pre-trained ArcFace model (ResNet-100 backbone) to convert face images into dense vector embeddings. The model was fine-tuned on a diverse dataset that includes Brazilian demographic distributions to minimize bias. Face detection and alignment (using MTCNN) are performed as preprocessing steps before embedding generation.

**Similarity Engine:** Computes cosine similarity between two embeddings. The match threshold is configurable per product type -- stricter for high-value loans, more lenient for micro-credit. Current default threshold: 0.72.

**Embedding Store:** Face embeddings are stored in PostgreSQL using the `pgvector` extension. This allows for efficient similarity searches and enables a future use case: detecting if the same face appears across multiple customer accounts (fraud detection via face deduplication).

**SQS Consumer / SNS Publisher:** Receives match requests from the `face-match-queue`, processes them, persists results, and publishes `FaceMatchCompleted` events.

## Data Flow

1. The Biometrics Capture Service publishes a match request to the `face-match-queue` containing: selfie S3 key, document photo S3 key, submission ID, and customer ID.
2. The service downloads both images from S3.
3. MTCNN detects and aligns faces in both images. If no face is detected in either image, the result is `NO_FACE_DETECTED`.
4. ArcFace generates a 512-dim embedding for each face.
5. Cosine similarity is computed. If above threshold, result is `MATCH`; if between threshold and threshold-0.1, result is `UNCERTAIN`; if below, result is `NO_MATCH`.
6. Embeddings and the match result are stored in PostgreSQL.
7. `FaceMatchCompleted` event is published to SNS.

## Integration Patterns

- **SQS (inbound):** Face match requests from the Biometrics Capture Service orchestrator.
- **SNS (outbound):** Match results consumed by the Biometrics Capture Service.
- **S3 (read):** Fetches selfie and document images.
- **PostgreSQL:** Persists embeddings (via `pgvector`) and match result history.
- **Customer Service (sync, optional):** May query customer data for logging and audit purposes.

## ADR-001: pgvector for Embedding Storage

**Context:** The team needed to store and query face embeddings. Options considered: (1) store as binary blobs in PostgreSQL, (2) use a dedicated vector database like Pinecone or Milvus, (3) use PostgreSQL with the `pgvector` extension.

**Decision:** Use `pgvector` in PostgreSQL. It supports vector storage, indexing (IVFFlat), and similarity search natively within the existing PostgreSQL infrastructure.

**Consequences:**
- No additional infrastructure component; embeddings live alongside match results in the same database.
- Enables future cross-account face deduplication queries using approximate nearest neighbor search.
- `pgvector` performance is adequate for the current scale (~50K embeddings). If the dataset grows to millions, migration to a dedicated vector database may be warranted.

## ADR-002: Product-Specific Match Thresholds

**Context:** A single global threshold for face matching created tension between fraud prevention (wants strict thresholds) and conversion (wants lenient thresholds to avoid false rejections). The product team observed that high-value secured loans justify stricter verification while micro-credit products need faster, more forgiving flows.

**Decision:** Make the similarity threshold configurable per product type. Thresholds are stored in a configuration table and passed as a parameter in the match request. Default: 0.72 (general), 0.78 (high-value), 0.68 (micro-credit).

**Consequences:**
- Flexibility to balance fraud risk and customer experience per product.
- Slightly more complex match logic, but the threshold is just a numeric comparison.
- Threshold values are reviewed quarterly by the fraud and product teams based on false positive/negative rates.

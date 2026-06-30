# Watchlist Service - Architecture Notes

## Component Diagram

```
                    +-------------------+
                    |  Kafka Consumer   |
                    | (biometrics_      |
                    |  validated)       |
                    +--------+----------+
                             |
                             v
+------------------+   +----+---------------+   +------------------+
| REST Controller  |-->| Screening Service  |-->| Customer Service |
| POST /screen     |   |                    |   | (REST client)    |
| GET /results/{id}|   +----+---------------+   +------------------+
+------------------+        |
                            v
                   +--------+---------+
                   | Matching Engine   |
                   |                   |
                   | +---------------+ |
                   | | Jaro-Winkler  | |
                   | | Similarity    | |
                   | +---------------+ |
                   | | Double        | |
                   | | Metaphone     | |
                   | +---------------+ |
                   +--------+----------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
        +---------+   +---------+   +---------+
        | OFAC    |   | PEP     |   | Internal|
        | Source   |   | Source   |   | Source   |
        +---------+   +---------+   +---------+
              |             |             |
              +-------------+-------------+
                            |
                            v
                   +--------+---------+
                   | Result Aggregator |
                   +--------+---------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
     +--------+---------+     +-----------+--------+
     | watchlist_db      |     | Kafka Producer     |
     | (PostgreSQL)      |     | (watchlist_checked) |
     +-------------------+     +--------------------+
```

## Data Flow

1. **Trigger:** The service is triggered either by a direct REST call or by the fraud pipeline processing a `biometrics_validated` event.

2. **Identity Resolution:** Customer identity data (full name, CPF, date of birth, nationality) is fetched from `customer-service` via synchronous REST call. This is the only external service dependency.

3. **Parallel Matching:** The matching engine runs the customer's identity against all active watchlist sources concurrently using Kotlin coroutines. Each source has an independent timeout (default 5s). If a source times out, the result is marked `PARTIAL` rather than failing the entire screening.

4. **Scoring:** Each potential match receives a confidence score based on:
   - Jaro-Winkler string similarity on name fields
   - Exact match on identifiers (CPF, passport number)
   - Phonetic similarity via Double Metaphone
   - Date of birth proximity

5. **Aggregation:** Results from all sources are aggregated. The highest confidence match determines the overall screening outcome (`CLEAR`, `POTENTIAL_MATCH`, `CONFIRMED_MATCH`).

6. **Persistence and Publishing:** The screening result is persisted to `watchlist_db` and a `watchlist_checked` event is published to Kafka.

## Integration Patterns

### Watchlist Source Ingestion

External watchlist sources (OFAC, UN, COAF PEP) are ingested via scheduled batch jobs. Each source has a dedicated parser that handles format-specific nuances. The ingestion follows an upsert pattern: existing entries are updated, new entries are inserted, and entries no longer in the source are soft-deleted.

The in-memory index used for fast matching is rebuilt after each source update. During the rebuild, the old index continues serving requests. The swap is atomic from the perspective of the matching engine.

### Event Integration

The service consumes `biometrics_validated` events indirectly - the fraud pipeline orchestrator routes the event and triggers a screening. The service publishes `watchlist_checked` events that are consumed by `fraud-scoring-service` as one of several fraud signals.

## ADR-001: In-Memory Watchlist Index for Matching Performance

**Status:** Accepted

**Context:** Initial implementation queried PostgreSQL for each screening request, resulting in screening latencies of 2-5 seconds due to the need for fuzzy string matching across hundreds of thousands of entries. The compliance SLA requires screening to complete within 3 seconds p99.

**Decision:** Build an in-memory index of all watchlist entries on service startup. The index stores normalized name variants and identifiers in a structure optimized for similarity search. The index is rebuilt when sources are updated.

**Consequences:**
- Screening latency dropped to 200-500ms p99
- Memory footprint increased to ~512MB for the full production dataset
- Startup time increased to 30-45 seconds while the index builds
- Each pod replica holds a full copy of the index, increasing overall cluster memory usage
- Source updates require an index rebuild, which takes ~10 seconds during which the old index is still serving

## ADR-002: Lowered Similarity Threshold After Compliance Incident

**Status:** Accepted

**Context:** In November 2024, a sanctioned individual with a transliterated name variant passed screening because the Jaro-Winkler similarity score was 0.81, below the threshold of 0.85. The compliance team flagged this as a critical gap.

**Decision:** Lower the similarity threshold from 0.85 to 0.78 and add phonetic matching (Double Metaphone) as a secondary matching strategy. Matches from either strategy are included in results.

**Consequences:**
- False positive rate increased from ~1% to ~3-5% of screenings
- Compliance team accepted the increased manual review workload
- The incident that triggered this change would now be caught (the transliterated name scores 0.81 on Jaro-Winkler and matches phonetically)
- Added monitoring for false positive rates to detect if further threshold tuning is needed

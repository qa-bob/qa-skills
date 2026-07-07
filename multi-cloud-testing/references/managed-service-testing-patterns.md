# Managed Service Testing Patterns

Testing patterns for the managed database, storage, and messaging services that differ meaningfully across AWS, GCP, and Azure — organized by service category with the CSP-specific equivalents side by side.

---

## Databases

| Category | AWS | GCP | Azure |
|---|---|---|---|
| Relational | RDS, Aurora | Cloud SQL, AlloyDB | Azure SQL, PostgreSQL Flexible Server |
| Wide-column / NoSQL | DynamoDB | Bigtable, Firestore | Cosmos DB |
| Distributed SQL | Aurora (regional), (no direct Spanner equivalent) | Cloud Spanner | Cosmos DB (with relational API) |

### Consistency testing

```python
# DynamoDB — default eventually-consistent read; must explicitly test
# the read-after-write timing gap this creates
def test_read_after_write_dynamodb():
    table.put_item(Item={"id": "123", "value": "updated"})
    # Eventually-consistent read may not immediately reflect the write —
    # test must either poll/retry or explicitly request strong consistency
    item = table.get_item(Key={"id": "123"}, ConsistentRead=True)
    assert item["Item"]["value"] == "updated"

# Cloud Spanner — strong consistency by default; a naive test written against
# Spanner's guarantees will falsely pass and then fail if ported to DynamoDB
# without re-checking the read-after-write assumption
```

Never assume a consistency guarantee transfers across a migration or a multi-cloud deployment — verify each managed database's actual default and configurable consistency options directly, and write the test against what's actually configured, not against what "databases generally do."

### Migration testing

Forward and backward migration correctness matters regardless of CSP, but the tooling differs: RDS/Aurora commonly use Flyway/Liquibase/native migration tooling against a Postgres/MySQL-compatible engine; Cloud SQL similarly for its Postgres/MySQL engines; Cosmos DB and DynamoDB, being schemaless, shift "migration" toward document-shape versioning and application-level handling of multiple document versions coexisting during a rollout, rather than a schema migration in the traditional sense.

---

## Object / Blob Storage

| Concern | AWS (S3) | GCP (Cloud Storage) | Azure (Blob Storage) |
|---|---|---|---|
| Public access check | Bucket policy + Block Public Access settings | IAM bindings for `allUsers`/`allAuthenticatedUsers` | Container public access level |
| Versioning/lifecycle | Versioning + lifecycle rules | Object versioning + lifecycle rules | Blob versioning + lifecycle management policies |
| Encryption at rest | SSE-S3/SSE-KMS | Google-managed or CMEK | Storage Service Encryption / customer-managed keys |

```python
# AWS — verify a bucket is genuinely not publicly readable
def test_s3_bucket_not_public(bucket_name):
    public_access_block = s3_client.get_public_access_block(Bucket=bucket_name)
    config = public_access_block["PublicAccessBlockConfiguration"]
    assert config["BlockPublicAcls"] and config["BlockPublicPolicy"]
    # Also verify no bucket policy independently grants public access —
    # Block Public Access alone doesn't cover every misconfiguration path
```

Write this same test intent against each CSP's actual API — the assertion "not publicly readable" is CSP-neutral, but the API call and the specific settings that can cause a false pass differ meaningfully per provider (S3 has more historically-common public-exposure misconfiguration vectors than the other two, which is itself useful CSP-specific context worth knowing when prioritizing this check).

---

## Messaging / Queues

| Category | AWS | GCP | Azure |
|---|---|---|---|
| Simple queue | SQS | Pub/Sub (also supports pub/sub patterns) | Service Bus Queues, Storage Queues |
| Pub/sub / fan-out | SNS + SQS | Pub/Sub | Service Bus Topics, Event Grid |
| Streaming | Kinesis | Pub/Sub (streaming mode), Dataflow | Event Hubs |

### What to test regardless of provider

- **At-least-once vs. exactly-once delivery semantics** — most managed queues default to at-least-once, meaning consumers must handle duplicate delivery; test that consumer logic is actually idempotent, not just assumed to be.
- **Dead-letter-queue routing** — verify a message that repeatedly fails processing actually lands in the DLQ after the configured retry count, and that DLQ contents are monitored/alertable, not silently accumulating unnoticed.
- **Ordering guarantees** — SQS FIFO queues, Pub/Sub ordering keys, and Service Bus sessions all provide ordering *only when explicitly configured for it* — verify the specific configuration in use actually provides the ordering the consumer logic assumes, rather than assuming standard/default queue behavior preserves order (it generally does not, across all three providers).

---

## Checklist

```
- [ ] Consistency-model tests verify the specific configured behavior of the actual managed service in use, not an assumption carried over from a different provider or a different service tier
- [ ] Public-exposure checks for storage use each provider's actual API/settings, not a single generic check
- [ ] Queue/messaging consumers are tested for idempotency under at-least-once delivery, unless the specific service/configuration genuinely guarantees exactly-once
- [ ] Ordering-guarantee tests verify the specific feature (FIFO/ordering keys/sessions) is both configured and actually enforced, not assumed from "it's a queue"
```

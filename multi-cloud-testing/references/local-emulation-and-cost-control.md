# Local Emulation & Cost Control

Testing against real cloud resources on every CI run is slow and expensive. Local emulation gets most of the confidence at a fraction of the cost and latency — knowing when emulation is good enough, and when it isn't, is the actual skill here.

---

## Emulator Options by CSP

| CSP | Emulator | Coverage |
|---|---|---|
| AWS | LocalStack | Broad service coverage (S3, DynamoDB, SQS, SNS, Lambda, and more) running as a single local container |
| GCP | Cloud SDK emulators (Firestore, Pub/Sub, Bigtable, Spanner each have dedicated emulators) | Per-service, not a single unified emulator like LocalStack |
| Azure | Azurite (Storage), Cosmos DB Emulator, Service Bus doesn't have an official emulator (use a real dev-tier resource or a third-party alternative) | Coverage is more uneven than AWS/GCP for some services |

```yaml
# docker-compose.test.yml — LocalStack example
services:
  localstack:
    image: localstack/localstack
    environment:
      - SERVICES=s3,dynamodb,sqs
    ports:
      - "4566:4566"
```

```python
# Point the SDK at the emulator instead of real AWS for tests
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://localhost:4566",  # LocalStack, not real AWS
    aws_access_key_id="test",
    aws_secret_access_key="test",
)
```

---

## What Emulation Is Good For, and Where It Diverges From Reality

| Good fit | Divergence risk |
|---|---|
| Fast unit/integration tests of application logic that happens to call a cloud SDK | IAM/permission behavior is often simplified or unenforced in emulators — a test that passes against an emulator with no real IAM restriction can pass despite a real IAM misconfiguration that would fail against the actual service |
| CI runs where cost and speed matter more than perfect fidelity | Some services' more exotic features (certain DynamoDB stream behaviors, certain Spanner transaction semantics) are incompletely emulated — check the emulator's documented feature coverage rather than assuming full parity |
| Local developer iteration without needing cloud credentials at all | Network-level behaviors (real latency, real throttling limits, cross-region behavior) aren't represented — anything testing resilience to real network conditions needs the real service or a dedicated fault-injection layer, not emulation |

**The practical rule:** use emulation as the default for fast iteration and most CI runs, but run a periodic (not necessarily per-PR) verification pass against real cloud resources for anything where IAM enforcement, exotic service features, or real network behavior are part of what's actually being tested. Treat an emulator-only test suite as validated-but-not-fully-proven until that periodic real-resource pass exists.

---

## Ephemeral Cloud Resources: Tagging and Cleanup

When real cloud resources genuinely are needed for a test run (periodic verification pass, or a feature the emulator can't cover), tag them clearly and make cleanup automatic and verified — not just intended:

```python
# Tag every test-provisioned resource distinctly and consistently
tags = {
    "purpose": "ephemeral-test",
    "created-by": "ci-pipeline",
    "ttl": "2h",  # informational; enforce via a separate reaper process
    "run-id": ci_run_id,
}
```

```bash
# A scheduled reaper job — independent of any specific test run succeeding or
# failing cleanly — that finds and removes anything tagged ephemeral-test
# past its TTL, as a backstop against a test run's own teardown step failing silently
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=purpose,Values=ephemeral-test \
  | jq '.ResourceTagMappingList[] | select(<age-check>)' \
  | <delete matched resources>
```

The reaper/backstop process matters independently of per-test teardown discipline (covered in `test-data-management`'s lifecycle guidance) — teardown steps fail sometimes (a crashed CI run, a script error), and without an independent backstop, those failures accumulate as real, ongoing cost rather than being caught and corrected automatically.

---

## Cost Visibility

Attribute cloud spend from testing infrastructure specifically, separate from production spend, so it's visible and manageable rather than an undifferentiated part of the overall bill:

- Use consistent tagging (as above) to filter cost-explorer/billing reports specifically for `purpose=ephemeral-test` spend
- Set a budget alert specifically on that tag filter, distinct from overall account budget alerts, so a runaway test-resource-creation bug (e.g. a CI retry loop that keeps provisioning without cleaning up) gets caught quickly rather than showing up as a surprise on the monthly bill
- Review test-infrastructure cost periodically as its own line item — it's easy for this to creep upward gradually as test suites grow, without anyone noticing until it's a meaningfully large number

---

## Checklist

```
- [ ] Emulation is the default for fast iteration; a periodic real-resource verification pass exists for anything IAM-sensitive or using less-completely-emulated features
- [ ] Every test-provisioned real cloud resource is tagged consistently and distinctly from production resources
- [ ] An independent reaper/backstop process removes ephemeral-test-tagged resources past their TTL, not relying solely on each test run's own teardown succeeding
- [ ] Test-infrastructure cost is tracked as its own visible line item with its own budget alert, not blended invisibly into overall spend
```

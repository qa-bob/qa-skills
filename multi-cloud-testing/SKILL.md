---
name: multi-cloud-testing
description: Test cloud-deployed systems across AWS, GCP, and Azure with CSP-aware patterns — managed service testing (databases, storage, queues), cloud-native security posture validation, infrastructure-as-code/policy-as-code testing, and local emulation for fast, cost-free iteration. Use this skill whenever the user is testing something running on a specific cloud provider, mentions Terraform/CloudFormation/Bicep, RDS/Cloud SQL/Azure SQL or other managed services, cloud security posture (Security Hub, SCC, Defender for Cloud), or asks how to test infrastructure without spinning up real (billable) cloud resources for every run.
---

# Multi-Cloud Testing

Testing a cloud-deployed system needs CSP-specific knowledge in a way that testing a plain application doesn't — "test the database layer" means writing genuinely different tests against Amazon RDS/Aurora/DynamoDB, GCP Cloud SQL/Spanner/Firestore, or Azure SQL/Cosmos DB, because each has different consistency models, different failure modes, and different native tooling for everything from security scanning to cost tracking. A test strategy written as if "the cloud" were one interchangeable thing either becomes AWS-specific by accident (breaking if the org multi-clouds or migrates) or stays so generic it never actually exercises the CSP-specific behavior that causes real production incidents.

This skill treats AWS, GCP, and Azure as peers throughout — every technique here has a concrete equivalent across all three, and the goal is CSP-neutral test *strategy* backed by CSP-specific test *implementation*, not a single provider baked in as the unstated default.

---

## When to reach for each part of this skill

| Situation | Do this |
|---|---|
| Testing a managed database, storage, or queue/messaging service | Read `references/managed-service-testing-patterns.md` |
| Verifying cloud security configuration (IAM, network exposure, compliance posture) | Read `references/cloud-security-posture-testing.md` |
| Testing Terraform/CloudFormation/Bicep before it's applied, or enforcing policy-as-code | Read `references/iac-testing-and-policy-as-code.md` |
| Need to test without provisioning real, billable cloud resources for every run | Read `references/local-emulation-and-cost-control.md` |
| None of the above — just getting oriented | Keep reading this file |

---

## Core Principle: CSP-Neutral Strategy, CSP-Specific Implementation

The test *intent* ("verify the database rejects a write that violates a foreign key constraint," "verify an S3-equivalent bucket isn't publicly readable") should be defined independent of any specific cloud provider. The test *implementation* that actually exercises that intent will differ by provider, because the underlying services differ:

```
Intent: "Object storage buckets must not be publicly readable by default"
├── AWS implementation: check S3 bucket policy + Block Public Access settings via
│                        AWS Config rule or a direct API check
├── GCP implementation: check Cloud Storage bucket IAM bindings for allUsers/allAuthenticatedUsers
└── Azure implementation: check Blob Storage container public access level setting
```

Structure test suites with this separation explicit — a shared, CSP-neutral test-intent definition, with CSP-specific implementation files as triplet siblings (`storage-exposure-aws.md` / `-gcp.md` / `-azure.md` style), selected by whichever CSP(s) the target actually deploys to. This is what makes a test strategy portable if the organization operates on multiple clouds, or migrates, without having to rewrite the underlying intent every time.

---

## Where CSP Differences Actually Bite

| Area | Why it's not interchangeable across CSPs |
|---|---|
| Consistency models | DynamoDB's default eventual consistency vs. Cloud Spanner's strong consistency vs. Cosmos DB's five selectable consistency levels — a test that assumes strong consistency will pass against one and flake against another |
| IAM models | AWS IAM policies, GCP IAM roles/bindings, and Azure RBAC + Azure AD are structurally different systems, not just different syntax for the same idea — least-privilege testing has to be written against each model's actual semantics |
| Native security tooling | AWS Security Hub/GuardDuty, GCP Security Command Center, Azure Defender for Cloud each surface different finding types and have different APIs for a test suite to query |
| Managed service failure modes | A managed database's failover behavior, connection-limit behavior, and cold-start latency differ meaningfully by provider and by specific service tier — testing "how does this degrade under load" needs provider-specific knowledge of what degradation actually looks like there |
| Cost model | Testing infrastructure cost accumulates differently — an idle RDS instance, an idle Cloud SQL instance, and an idle Azure SQL instance are billed differently, which affects how aggressively ephemeral test infra needs to be torn down (see `local-emulation-and-cost-control.md`) |

---

## Decision Tree

```
What CSP(s) does the target actually deploy to?
├── Single CSP, known — go straight to that CSP's implementation in whichever
│   reference file matches the test category needed
├── Multi-cloud (same workload on more than one CSP) — write the CSP-neutral
│   intent once, implement the triplet-sibling pattern for each CSP in scope
├── CSP-agnostic target (containerized, no CSP-specific managed services used) —
│   most of this skill doesn't apply; test as you would test any containerized
│   service, and revisit this skill only if CSP-specific services get introduced later
└── Unknown / to be determined — default to local emulation
    (references/local-emulation-and-cost-control.md) until the deployment target
    is decided, so test-writing isn't blocked on an infrastructure decision
```

---

## Checklist

```
- [ ] Test intent is defined CSP-neutrally, with CSP-specific implementation kept as separate, clearly-labeled variants
- [ ] Consistency-model assumptions are verified against the actual managed service's real guarantees, not assumed to match another CSP's default
- [ ] Security posture tests use each CSP's native tooling/APIs rather than a generic check that misses provider-specific finding types
- [ ] Ephemeral test infrastructure is tagged and torn down per CSP's own cost/cleanup tooling conventions
- [ ] If the target might migrate or expand to a second CSP later, the CSP-neutral/CSP-specific separation is already in place rather than retrofitted after the fact
```

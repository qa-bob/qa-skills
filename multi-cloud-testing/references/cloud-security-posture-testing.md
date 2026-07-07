# Cloud Security Posture Testing

Testing that cloud infrastructure configuration itself doesn't create security exposure — distinct from application-level security testing (covered in the separate `security-testing` skill). This is about IAM, network exposure, and compliance posture at the infrastructure layer, using each CSP's native tooling.

---

## Native Security Tooling Per CSP

| Capability | AWS | GCP | Azure |
|---|---|---|---|
| Aggregated security findings | Security Hub | Security Command Center | Defender for Cloud |
| Threat detection | GuardDuty | Security Command Center (Event Threat Detection) | Microsoft Sentinel |
| Config/compliance drift | AWS Config | Security Command Center (posture management) | Azure Policy |
| Sensitive data discovery | Macie | Cloud DLP | Microsoft Purview |
| IAM analysis | IAM Access Analyzer | IAM Recommender | Azure AD access reviews |
| Web app firewall | AWS WAF | Cloud Armor | Azure Front Door WAF / Application Gateway WAF |

Testing against these tools means querying their findings APIs as part of a test/CI run, not just reviewing dashboards manually — treat "no unresolved high/critical finding in [native tool]" as an automatable, CI-gateable assertion the same way a SAST/DAST gate works in `security-testing`.

```python
# AWS example — fail a pipeline if Security Hub has unresolved critical findings
def test_no_critical_security_hub_findings():
    findings = securityhub_client.get_findings(
        Filters={
            "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}],
            "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}],
        }
    )
    assert len(findings["Findings"]) == 0, f"Unresolved critical findings: {findings['Findings']}"
```

---

## IAM / Least-Privilege Testing

Each provider's IAM model is structurally different, so "test least privilege" means different concrete checks per provider:

| Provider | What to check |
|---|---|
| AWS | IAM policies attached to roles/users — look for wildcard actions (`"Action": "*"`) or wildcard resources (`"Resource": "*"`) on anything beyond genuinely-broad service roles; use IAM Access Analyzer to detect unused permissions |
| GCP | IAM role bindings at the project/folder/org level — check for overly broad predefined roles (e.g. `roles/editor` or `roles/owner` granted where a narrower custom/predefined role would suffice); use IAM Recommender's actual-usage-based suggestions |
| Azure | RBAC role assignments — check for `Owner`/`Contributor` granted at a broad scope (subscription-level) where a narrower resource-group or resource-level assignment with a more specific built-in role would suffice |

A useful automatable heuristic across all three: flag any principal (user, service account, or role) with write/administrative access to a broader scope than what its actual usage pattern (from CloudTrail / Cloud Audit Logs / Azure Activity Log) shows it needs — over-provisioned access that's never actually exercised is the most common and most fixable finding in this category.

---

## Network Exposure Testing

| Check | AWS | GCP | Azure |
|---|---|---|---|
| Overly permissive ingress rules | Security Group rules allowing `0.0.0.0/0` on sensitive ports | Firewall rules with source range `0.0.0.0/0` on sensitive ports | Network Security Group rules with source `Any`/`Internet` on sensitive ports |
| Public IP exposure on resources that shouldn't have one | EC2/RDS with a public IP where a private subnet + NAT/VPN would suffice | Compute Engine/Cloud SQL with a public IP where Private Service Connect/VPC peering would suffice | VMs/managed databases with a public endpoint where Private Link/VNet integration would suffice |

Automate a periodic scan for "sensitive port + wide-open source range" combinations (SSH/RDP/database ports open to `0.0.0.0/0` or equivalent) — this specific misconfiguration is disproportionately common and disproportionately exploited, making it worth a dedicated, always-on check rather than only catching it during a periodic manual review.

---

## Compliance Posture Testing

For regulated environments (see `test-data-management`'s regulatory context for the data-handling side of this), each provider offers compliance-framework-mapped posture checks:

- AWS Config conformance packs (CIS AWS Foundations, PCI-DSS, HIPAA-eligible service usage)
- Security Command Center's compliance dashboards (CIS GCP, PCI-DSS)
- Azure Policy's regulatory compliance initiatives (CIS Azure, PCI-DSS, HIPAA/HITRUST)

Wire these into the same CI-gating pattern as the other checks in this file — a compliance posture regression (a newly non-compliant resource) should be caught the same way a new security finding is caught, not discovered only during a periodic audit cycle.

---

## Checklist

```
- [ ] Unresolved high/critical findings from the CSP's native security aggregator are a CI/pipeline gate, not just a dashboard someone occasionally checks
- [ ] IAM least-privilege checks use each provider's actual-usage-based recommender (Access Analyzer / IAM Recommender / access reviews), not just a static policy review
- [ ] Network exposure scans specifically flag sensitive-port + wide-open-source-range combinations as a dedicated, always-on check
- [ ] Compliance posture checks (CIS/PCI/HIPAA mappings) run continuously, not only during periodic manual audits
```

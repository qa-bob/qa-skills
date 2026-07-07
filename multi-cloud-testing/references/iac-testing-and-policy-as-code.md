# IaC Testing & Policy-as-Code

Testing infrastructure-as-code before it's applied catches misconfigurations before they ever become a real (billable, exploitable, or compliance-violating) resource — the cloud-infrastructure equivalent of catching a bug at the SAST layer instead of in production.

---

## Levels of IaC Testing

```
1. Syntax/validation    — is the Terraform/CloudFormation/Bicep syntactically valid?
                           (terraform validate, cfn-lint, bicep build)
2. Static security scan — does the plan contain known-bad patterns (public S3 buckets,
                           overly permissive security groups, unencrypted resources)?
3. Policy-as-code check — does the plan comply with org-specific rules that go beyond
                           generic security scanning (e.g. "all resources must be tagged
                           with a cost-center", "no resources outside approved regions")?
4. Plan-diff review     — does the actual planned change match intent (e.g. does a
                           "small config tweak" PR unexpectedly show a plan to destroy
                           and recreate a stateful resource)?
```

Levels 1-3 are automatable and belong in CI, gating the PR before apply. Level 4 benefits from automation too (surfacing destroy/recreate actions prominently) but usually still wants a human look before anything stateful gets applied.

---

## Static Security Scanning for IaC

| Tool | Coverage |
|---|---|
| Checkov | Broad multi-cloud coverage (Terraform, CloudFormation, ARM/Bicep, Kubernetes manifests), large built-in rule set |
| tfsec / Trivy (IaC mode) | Terraform-focused, fast, good default ruleset |
| Terrascan | Multi-cloud, policy-as-code via OPA under the hood |

```yaml
# CI wiring — same shape as the SAST pattern in security-testing
- name: IaC security scan
  run: checkov -d ./infra --framework terraform --compact
```

Apply the same baseline-then-gate-on-new-findings discipline described in `security-testing`'s SAST guidance — a first scan against existing infrastructure code will often surface a large volume of pre-existing findings; baseline those and gate CI on new findings introduced by the current change, rather than blocking every PR on the full historical backlog.

---

## Policy-as-Code Beyond Generic Security Scanning

Generic scanners catch generic bad patterns. Org-specific rules — the ones that encode this organization's actual standards — need policy-as-code tooling that can express arbitrary custom logic:

```rego
# Open Policy Agent (OPA) / Rego example — enforce that every resource
# is tagged with a cost-center, an org-specific rule no generic scanner knows about
package terraform.tagging

deny[msg] {
    resource := input.resource_changes[_]
    resource.change.actions[_] == "create"
    not resource.change.after.tags.cost_center
    msg := sprintf("Resource %v is missing required tag: cost_center", [resource.address])
}
```

```bash
# Run against a terraform plan's JSON output
terraform show -json plan.tfplan > plan.json
opa eval --input plan.json --data policy.rego "data.terraform.tagging.deny"
```

Cloud-native equivalents exist too: AWS Service Control Policies / Config custom rules, GCP Organization Policy constraints, Azure Policy custom definitions — these enforce similar org-specific rules at the platform level as a backstop even if something bypasses the pre-apply IaC check, which is worth having as defense-in-depth rather than relying on pre-apply testing alone.

---

## Plan-Diff Review for Stateful Resources

The most consequential IaC mistakes usually aren't syntax errors — they're a plan that does something more destructive than the author intended, often because of how the underlying provider handles certain attribute changes (some attribute changes force a resource replacement rather than an in-place update, which isn't always obvious from the diff of the source `.tf`/`.bicep`/CloudFormation file itself).

```bash
# Always review the actual plan output, not just the source diff
terraform plan -out=plan.tfplan
terraform show plan.tfplan | grep -E "^\s*[-+~]" 
# Specifically look for "# forces replacement" annotations near stateful
# resources (databases, storage with data) — this is the single highest-value
# thing to catch before apply
```

Automate a check that flags (and requires explicit human acknowledgment for) any plan showing a destroy/recreate on a resource tagged as stateful/data-bearing — this narrow, targeted check catches a disproportionate share of the worst IaC incidents relative to its implementation cost.

---

## Checklist

```
- [ ] IaC changes go through syntax validation, static security scanning, and policy-as-code checks before apply — not just a manual review of the source diff
- [ ] Static scan findings are baselined, with CI gating only on new findings, to avoid an unmanageable volume on day one
- [ ] Org-specific rules (tagging, region restrictions, naming conventions) are enforced via policy-as-code, not left to scanner defaults that don't know this org's standards
- [ ] Plans showing destroy/recreate on stateful/data-bearing resources are explicitly flagged and require human acknowledgment before apply
- [ ] Platform-level policy enforcement (SCPs/Org Policy/Azure Policy) exists as defense-in-depth, not as the only enforcement layer
```

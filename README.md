# QA & Testing Skills

A collection of Claude Skills covering software testing and quality engineering disciplines that are under-represented in the broader Claude Skills marketplace ecosystem — written from real QA/testing practice rather than generated as generic advice.

Each skill lives in its own directory with a `SKILL.md` entry point and, where the topic is deep enough to need it, a `references/` folder of supporting material loaded on demand.

## Skills in this collection

| Skill | Covers |
|---|---|
| [`security-testing/`](./security-testing/) | OWASP Top 10 test case design, SAST/DAST CI integration, secrets & container scanning, penetration-test planning |
| [`llm-evaluation/`](./llm-evaluation/) | Behavioral & consistency testing, adversarial/prompt-injection & safety testing, RAG retrieval & generation metrics, LLM-as-judge & benchmark design |
| [`test-data-management/`](./test-data-management/) | Synthetic data generation, PII anonymization/masking of production-derived data, referential integrity across fixtures, test data lifecycle & environment management |
| [`contract-testing/`](./contract-testing/) | Consumer-driven contract testing with Pact — consumer-side test authoring, provider verification, Pact Broker & can-i-deploy CI workflow, message/async contracts |
| [`multi-cloud-testing/`](./multi-cloud-testing/) | CSP-aware managed service testing (AWS/GCP/Azure), cloud security posture testing, IaC & policy-as-code testing, local emulation & cost control |

Remaining backlog: test pipeline observability and test documentation/closure reporting.

## Using a skill

1. Open the skill's directory and copy its `SKILL.md` (and `references/` folder, if present).
2. Paste into `~/.claude/skills/<skill-name>/`.
3. Restart Claude Code / Claude Desktop.

## License

MIT — see [LICENSE](./LICENSE).

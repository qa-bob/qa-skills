# QA & Testing Skills

A collection of Claude Skills covering software testing and quality engineering disciplines that are under-represented in the broader Claude Skills marketplace ecosystem — written from real QA/testing practice rather than generated as generic advice.

Each skill lives in its own directory with a `SKILL.md` entry point and, where the topic is deep enough to need it, a `references/` folder of supporting material loaded on demand.

## Skills in this collection

| Skill | Covers |
|---|---|
| [`security-testing/`](./security-testing/) | OWASP Top 10 test case design, SAST/DAST CI integration, secrets & container scanning, penetration-test planning |
| [`llm-evaluation/`](./llm-evaluation/) | Behavioral & consistency testing, adversarial/prompt-injection & safety testing, RAG retrieval & generation metrics, LLM-as-judge & benchmark design |
| [`test-data-management/`](./test-data-management/) | Synthetic data generation, PII anonymization/masking of production-derived data, referential integrity across fixtures, test data lifecycle & environment management |

More to follow — consumer-driven contract testing is next.

## Using a skill

1. Open the skill's directory and copy its `SKILL.md` (and `references/` folder, if present).
2. Paste into `~/.claude/skills/<skill-name>/`.
3. Restart Claude Code / Claude Desktop.

## License

MIT — see [LICENSE](./LICENSE).

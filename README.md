# HAIEC — Deterministic Control Proof for Agentic Systems

How do you prove what an AI agent actually did—and whether it followed the rules?

This project supports the walkthrough video for **HAIEC: Deterministic Control Proof for Agentic Systems**, created for the **TM Forum Innovate Americas 2026 Hackathon**.

It demonstrates how Customer, IT, and Network agents connect through a governed gateway, how LogSense reconstructs runtime evidence, and how HAIEC evaluates that evidence against frozen control policies.

## Controls covered

- **C7 — Event Recording:** Nine of ten required evidence categories established; missing negotiation evidence remains visible.
- **C9 — Performance Drift:** Both assessed runs stayed within the frozen threshold; an eligible breach was not established.
- **C16 — Token Spend:** A passing run and a breaching run evaluated against the same 60,000-token cap.

The walkthrough also covers agent architecture, Cedar authorization, Amazon Bedrock guardrails, CloudWatch evidence, and reproducible control verification.

> The C16 assessment detects overspending after execution; it does not demonstrate prevention during the run.

**Same evidence. Same policy. Same verdict. Unknown stays unknown.**

## Links

- Explore the project and evidence: https://subodhkc.com/tm-forum
- Judge report: https://subodhkc.com/tm-forum/assets/TMF_2026_HAIEC_JUDGE_REPORT.html
- Learn more: https://haiec.com

## Team

Subodh KC • Kushal Gautam • Abhishek Yadav • Atul Mishra

---

Which control would you prioritize before deploying AI agents in your organization?

#AgenticAI #AIGovernance #TMForum #HAIEC #AISecurity

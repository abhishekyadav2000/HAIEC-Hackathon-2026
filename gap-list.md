# Gap list — selected control C16 (ACN-COST-001)

Open limitations preserved, not greenwashed:
- RUN_START is operator-declared; no platform run-start signal exists.
- 64KB per-record ingest bound: runaway run (2.83M tokens) never reached HAIEC — no verdict manufactured.
- Token-ceiling policy `modaas-ceiling` uses local rate-limit `tokens:` field, which per agentgateway
  docs counts REQUESTS not LLM tokens — named "token ceiling" does not bound LLM tokens. Observed:
  91 calls, 2,828,243 gateway tokens, zero token-based rejects.
- C7 negotiation event never observed in 7+ runs (capability exists; model-decisional).
- C9 (AIA-ARC-006) UNESTABLISHED — metric source captured, KPI candidate PROPOSED only.
- Audit-store records are agent self-written under a shared token; corroborated by
  platform-attested /modaas/evidence action_ids and gateway usage figures.
- Flat PDP identity: all decisions show ServiceIdentity::"bootstrap-key".
- Agent image spec drift: CR desired digest ahead of operator-handled digest;
  running runtime = handled digest (no attestation drift proven).
- ServiceNow AICT: AWS_PRECHECK_COMPLETE + FACILITATOR_ACTIVATION_PENDING (B-008 open).

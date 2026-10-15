# Evidence file

Covers controls 7 (event recording), 16 (per-run token cap) and 9 (drift in agent duration) on the TM Forum MoDaaS lab: the named pair for 7 and 16, the named pair for 9
(assessed runs after the HAIEC control 9 freeze), one supporting run, and three rehearsal runs of the organizers' own agents for comparison (controls 7 and 16 only: our control 9 baselines are for our agents).
Every file is plain JSON, JSON lines or text. Nothing here needs our tools to open.

## Runs
- `fault-1791219795-d2c401`: PASS (named, controls 7 and 16). Control 16: SATISFIED   29,449 tokens against a limit of 40,000 (10,551 under). Control 7: SATISFIED   10 of 10 declared events recorded (100% against 100% required); 0 of 28 gaps over 30,000 ms (largest 14,000 ms, ledger time is whole seconds).
- `fault-1791214684-1213da`: BREACH (named, controls 7 and 16). Control 16: NOT SATISFIED   50,853 tokens against a limit of 40,000 (10,853 OVER). Control 7: NOT SATISFIED   9 of 10 declared events recorded (90% against 100% required); MISSING NEGOTIATION by network-resolution-agent; 0 of 26 gaps over 30,000 ms (largest 6,000 ms, ledger time is whole seconds).
- `fault-1791211739-11dd2a`: supporting: the pass of the HAIEC control 16 pair. Control 16: SATISFIED   33,855 tokens against a limit of 40,000 (6,145 under). Control 7: NOT SATISFIED   9 of 10 declared events recorded (90% against 100% required); MISSING NEGOTIATION by network-resolution-agent; 0 of 28 gaps over 30,000 ms (largest 14,000 ms, ledger time is whole seconds).
- `fault-1791276114-723d80`: PASS (named, control 9): assessed run S3, our chain. Control 16: SATISFIED   34,393 tokens against a limit of 40,000 (5,607 under). Control 7: SATISFIED   10 of 10 declared events recorded (100% against 100% required); 0 of 33 gaps over 30,000 ms (largest 4,000 ms, ledger time is whole seconds). Control 9: SATISFIED   worst agent haiec-network-agent +60.2% against a limit of +100%; 0 of 3 windows over (tolerance 0%).
- `fault-1791275895-048d48`: BREACH (named, control 9): assessed run S1, our chain. Control 16: SATISFIED   39,733 tokens against a limit of 40,000 (267 under). Control 7: SATISFIED   10 of 10 declared events recorded (100% against 100% required); 0 of 33 gaps over 30,000 ms (largest 14,000 ms, ledger time is whole seconds). Control 9: NOT SATISFIED   worst agent it-resolution-agent +286.0% against a limit of +100%; 1 of 3 windows over (tolerance 0%).
- `fault-1791276013-f00536`: assessed run S2, our chain. Control 16: SATISFIED   33,679 tokens against a limit of 40,000 (6,321 under). Control 7: SATISFIED   10 of 10 declared events recorded (100% against 100% required); 0 of 30 gaps over 30,000 ms (largest 6,000 ms, ledger time is whole seconds). Control 9: SATISFIED   worst agent haiec-network-agent +22.4% against a limit of +100%; 0 of 3 windows over (tolerance 0%).
- `fault-1791276579-6fc0f5`: comparison: S1, organizers' agents (rehearsal; controls 7 and 16 only). Control 16: NOT SATISFIED   45,136 tokens against a limit of 40,000 (5,136 OVER). Control 7: NOT SATISFIED   9 of 10 declared events recorded (90% against 100% required); MISSING NEGOTIATION by network-resolution-agent; 0 of 31 gaps over 30,000 ms (largest 14,000 ms, ledger time is whole seconds).
- `fault-1791276692-4c145c`: comparison: S2, organizers' agents (rehearsal; controls 7 and 16 only). Control 16: SATISFIED   36,622 tokens against a limit of 40,000 (3,378 under). Control 7: NOT SATISFIED   9 of 10 declared events recorded (90% against 100% required); MISSING NEGOTIATION by network-resolution-agent; 0 of 27 gaps over 30,000 ms (largest 6,000 ms, ledger time is whole seconds).
- `fault-1791276788-49f372`: comparison: S3, organizers' agents (rehearsal; controls 7 and 16 only). Control 16: SATISFIED   20,112 tokens against a limit of 40,000 (19,888 under). Control 7: NOT SATISFIED   9 of 10 declared events recorded (90% against 100% required); MISSING NEGOTIATION by network-resolution-agent; 0 of 21 gaps over 30,000 ms (largest 4,000 ms, ledger time is whole seconds).

## What is in `runs/<run id>/`
| File | What it is |
|---|---|
| `tmf639.json`, `cr-status.json`, `approvals.json`, `gateway-log.json`, `registry.json`, `otel-traces.json` | the six standard surfaces from the organizers' `pull-evidence.sh`. `gateway-log.json` is joined on the trace id the run's ledger names (the gateway lines carry no run id). `otel-traces.json` is the audit ledger's own chain, as the script explains |
| `ledger.jsonl` | the run's full audit-store records (one JSON object per line, in order); the organizers' `/timeline` view cuts text at 200 characters, this does not |
| `timeline.txt` | the organizers' timeline view of the same run |
| `judge-control-16.txt`, `judge-control-7.txt`, `judge-control-9.txt` | the verdict and the records behind it, from `python -m assurance.judge`, against `register.yaml` (`judge-control-9.txt` exists for our chain's control 9 runs) |
| `durations.json` | the CloudWatch AgentCore `Duration` per-minute averages control 9 used (agent, minute, average ms, invocations), so the verdict replays offline |
| `auditor-six-questions.txt` | the six auditor questions for the run (who, what, authorized, approved, integrity, reconstruct) |
| `gateway.log` | the gateway's raw lines for the run (rebuilt from `gateway-log.json`), the input the tool reads for control 16 |
| `haiec_bundles/` | the LogSense bundles we generated for HAIEC (`verdict` is null; the verdict belongs to HAIEC) |

## The control test tool and the recorded demo
`tool/` is a runnable copy of the tool (standard library only; see `tool/README.md`). `demo/transcript.txt` is a recording of eight questions put to it
offline, from the files in this folder: control 16 and control 7 on the passing and breaching run, both controls over a time window, and control 9 on the passing
and breaching assessed run. Each run folder has a `gateway.log` (the gateway's own lines for that run), so
the tool replays without lab access.

## Reproduce a verdict
```
export AWS_PROFILE=hackathon KUBECONFIG=~/hackathon.kube
python -m assurance.judge --control 16 --run fault-1791214684-1213da          # live
python -m assurance.judge --control 16 --run <id> --records runs/<id>/ledger.jsonl --gateway-log <gateway log>   # from saved files
python -m assurance.judge --control 7 --window 2026-10-05T14:40:00Z 2026-10-05T17:10:00Z --actor-prefix haiec  # a time window
```

## HAIEC results on our system (44f861cf-9c09-43e5-9388-997b66649542), read from the tenant on 2026-10-05
| Control | Run | Result | Result id |
|---|---|---|---|
| 16 (policy c16-cap-v2, 40,000) | fault-1791211739-11dd2a | SATISFIED, 33,855 tokens | ctr-48dd8310c0f8c3918e2908d2493b73ba8a440656 |
| 16 | fault-1791214684-1213da | NOT SATISFIED, 50,853 tokens, SPEND_CAP_EXCEEDED | ctr-44295c6586b99f3e657fed47d0c2515ca5c4c6f2 |
| 7 (policy c7-thresholds-v1) | fault-1791219795-d2c401 | SATISFIED, 10 of 10 events | ctr-d4d6430a502a8383a9c043640757700fb2bd96a7 |
| 7 | fault-1791214684-1213da | NOT SATISFIED, 9 of 10 | ctr-7f2772a2c5a5635c94915e2b7ff9087f67bc4a2e |
| 7 | fault-1791211739-11dd2a | NOT SATISFIED, 9 of 10 | ctr-e8bc60b3a2e3fa394fa57a17cc92325a541abdbf |

| 9 (policy c9-duration-thresholds-haiec-v1, frozen 2026-10-06T08:23:41Z, policy 13c07652-30e1-46bf-855f-fb3cbde51dcf) | fault-1791275895-048d48, fault-1791276114-723d80 | no HAIEC Control Test result yet: no bundle has been uploaded for these runs | none |

For control 9 the policy is frozen and eligible in HAIEC, but no scored result exists on our system; the verdicts for control 9 in this folder come from our tool (`judge-control-9.txt`).
The control 16 bundle of `fault-1791219795-d2c401` is in `haiec_bundles/` but its HAIEC result is not persisted yet; `judge-control-16.txt` has the verdict from our tool.
HAIEC closed three control 7 breaches with the d2c401 result (2026-10-05T17:11:31Z).

## Redaction
The lab's shared audit write token appears in plain text in the agents' environment variables (`cr-status.json`, as exported by the organizers' script).
It is removed from every exported file (runs/fault-1791211739-11dd2a/cr-status.json, runs/fault-1791214684-1213da/cr-status.json, runs/fault-1791219795-d2c401/cr-status.json, runs/fault-1791275895-048d48/cr-status.json, runs/fault-1791276013-f00536/cr-status.json, runs/fault-1791276114-723d80/cr-status.json, runs/fault-1791276579-6fc0f5/cr-status.json, runs/fault-1791276692-4c145c/cr-status.json, runs/fault-1791276788-49f372/cr-status.json). Nothing else was edited.

## Other documents
`docs/S2_AMBIGUITY_SWITCH.md` (the S2 evidence and the grader's behaviour, with reproduction commands) and `docs/FINDINGS.md` (findings and re-tests).

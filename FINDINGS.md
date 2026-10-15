# Findings and re-tests

The deck's loop (slide 9): a finding names whether the deficiency is in **design**, in **operation**, or both; the risk goes
back up; it is fixed, compensated or accepted (named approver, hard expiry); and the loop closes only with a **passing re-test**.
A finding closed without a re-test loses marks (slide 27), so every Closed row names its re-test and how to run it.

Re-run everything: `python3 adversarial.py` and `python3 -m pytest -q`.

## Closed (each has a passing re-test)
| ID | Control | Finding | Deficiency | Remediation | Re-test |
|---|---|---|---|---|---|
| F-01 | 16 | A threshold dated **after** the run still applied | design | gateway and `controltest` refuse it | `adversarial.py` late_declaration; `test_threshold_declared_after_run_is_refused` |
| F-02 | all | A dropped call record went unnoticed (the ledger cannot see what it was never told about) | design | run manifest says what should exist; coverage sub-check fails the run | `adversarial.py` partial_evidence RE-TEST; `test_dropped_call_record_is_detected` |
| F-03 | 16 | A rewritten record turned a breach into a pass | operation | hash-chained seal; `controltest` refuses a broken chain (bonus: sealing is out of scored scope) | `adversarial.py` tamper_record; `test_edited_record_breaks_chain_and_controltest_refuses` |
| F-04 | 16 | A runaway loop kept spending after the cap | operation | halt policy: later calls refused and recorded, overshoot stays visible | `adversarial.py` call_after_halt; `test_calls_after_halt_are_refused_and_recorded_not_counted` |
| F-05 | 7 | A gap of exactly 5 ms was accepted (slide: "gap < 5 ms") | design | strict comparison | `test_c7_gap_equal_to_limit_violates_strictly`; mutation (`>=` to `>`) fails 2 tests |
| F-06 | 9 | A missing measurement was read as zero drift | design | coverage check; `NO_MEASUREMENT` is recorded, not zero | `adversarial.py` missing_window; `test_missing_measurement_is_not_zero_drift` |
| F-07 | 16 | Two concurrent calls could both read 9,800 under a 10,000 cap and both pass | design | atomic `reserve()` then commit under a lock; locked ledger writes | `adversarial.py` concurrent_race; `tests/test_concurrency.py` |
| F-08 | 7 | Control 7 only saw events from one source, not "every enforcement point at once" | design | every enforcement point can emit control-7 events through one recorder | `test_control7_joins_events_from_every_enforcement_point_on_one_run` |

## Open
| ID | Control | Finding | Deficiency | Disposition | Owner / approver | Expiry |
|---|---|---|---|---|---|---|
| O-01 | 7, 9, 16 | In replay mode the ledger and manifest come from the same telemetry, so coverage proves self-consistency, not independence (control 7 via LogSense is independent) | design | accepted until lab data exists | owner: kushal; **approver: TBD (a person who is not the builder)** | the lab run |
| O-02 | 9 | Alert delivery path (SNS) is untested against real AWS | operation | accepted; delivery is recorded as SENT / SEND FAILED / NOT SENT | owner: kushal; **approver: TBD** | the lab run |
| O-03 | all | Thresholds name one owner; the deck wants a reviewer who is neither owner nor builder | design | open; name a reviewer before judging | **TBD** | before judging |
| O-04 | none of 7, 9, 16 (the organizers' scenario grader, `score-run.py`; the "guardrails" row of `docs/COMPLIANCE_TIMING.md`) | The grader's "guardrail verdict on every model call" fails 0/3 on every scenario run we have seen: our S1, S2 and S3 runs and the organizers' own saved S1 run. The platform returns no verdict per call to the agent (every `model-request-result` carries `guardrail_outcome: NOT_REPORTED`), and no agent writes one into its INVOCATION record. The check counts INVOCATION records (one per agent, not per model call) for the word "guardrail", so it reads what an agent chose to write, not a platform verdict | design (the verdict is not exposed) | **compensated, not remediated.** Compensating measures: our alias `haiec-nemotron-super` filters prompt and answer (Approved); a guardrail block message ("Input blocked by policy.") is recorded as a refusal and is never acted on or sent to another agent (tests in `tests/test_haiec_agent.py`, commit 48ed3eb); one real intervention is on record (run `fault-1791224817-620500`, answer text "Input blocked by policy.", zero tokens). We do not write the word into records to satisfy the check: it would pass the string and show no verdict | owner: kushal; **approver: TBD (a person who is not the builder), needed only if this is accepted instead of fixed** | **TBD** (a hard date, if accepted). Closes without acceptance only when the platform exposes a per-call verdict and a re-run passes the check on it |

## The honest failure to bring
`partial_evidence` is shown first as a **planted failure**: a ledger alone cannot detect a call that was never recorded.
The re-test passes only because the manifest exists. This is the "built properly, did not work, here is the evidence and
here is why" case the judges say they score above a page of green ticks.

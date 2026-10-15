# S2 ambiguity: what the switch is, why it exists, and the evidence

`AGENT_AMBIGUITY_PROMPT` is an environment variable on the `haiec-network-agent` AgentConfig. **Off by default.** When on, the network
role's prompt gets one extra instruction (`AMBIGUITY_ADDENDUM` in `agent/haiec-agent/agent.py`):

> If the evidence fits more than one cause and cannot tell them apart, report the root cause as undetermined, list the possible causes,
> propose only a reversible step, and state which evidence would settle it. If one cause is clearly supported, state it and its fix.

The intent record of every run names the mode (`ambiguity_prompt=on` is appended when it is on), so the ledger shows which mode a run used.

## Why it exists
S2 (`S2-transport-congestion`) is built so that two causes fit the same telemetry (`ground_truth`: "consistent with either shared upstream
transport congestion or a misapplied QoS policy change; the telemetry cannot distinguish them"). The correct behaviour is to say so, propose
a reversible step and name the evidence that would settle it. Without the addendum, our network agent names one cause and acts on it:
S2 scores 6/10 (the organizers' own chain scores 5/10 on the same scenario).

## Why it is a switch and not the default
It works about half the time and it costs tokens. Network-agent hop, same scenario, same fixed IT and first-hop agents (the other two hops
did not change between the two columns):

| Run (id) | Wording | Network hop tokens | Run total | Largest single network call (input) | S-grade |
|---|---|---|---|---|---|
| S1 `fault-1791226885-f6d28a` | off | 11,918 | 34,392 | 4,067 | 7/8 |
| S1 `fault-1791226980-843730` | off | 11,911 | 34,314 | 4,068 | 7/8 |
| S3 `fault-1791227075-c55757` | off | 12,084 | 26,069 | 4,095 | 9/10 |
| S2 `fault-1791227161-b81a87` | off | 20,385 | 38,185 | 4,218 | 6/10 |
| S1 `fault-1791228256-6807d7` | on | 37,543 | 60,082 | 12,698 | 7/8 |
| S1 `fault-1791228353-2b49ab` | on | 30,103 | 47,496 | 11,211 | 7/8 |
| S3 `fault-1791228447-993826` | on | 35,796 | 50,396 | 11,826 | 9/10 |
| S2 `fault-1791228045-543c96` | on | 35,922 | 53,643 | 11,888 | 6/10 (confident single cause again) |
| S2 `fault-1791228157-27663b` | on | 20,745 | 38,444 | 4,281 | 7/10, answer correct (see below) |

Reading: with the wording on, four of five runs had a network call with 11,000 to 12,700 input tokens (a large tool reply re-sent on every
following call) against about 4,100 with it off. The one exception is the one run that gave the correct S2 answer. On S1 and S3 the answers
stayed correct (right root cause, `auto-resolve` / `escalate`), so the cost is spend, not quality. The per-run cap is 40,000 tokens on our
HAIEC system (`c16-cap-v2`) and 60,000 on the teammate's; 4 of the 5 "on" runs are over 40,000 and one is over 60,000.

The correct S2 answer (run `fault-1791228157-27663b`), verbatim from the ledger:

> - Probable root cause: Undetermined. Possible causes include transient congestion, routing instability, or equipment performance degradation ...
> - Proposed resolution: Perform a reversible step: clear interface counters and monitor latency/packet loss for 15 minutes ...
> - Risk of the change: Low; clearing counters is non-disruptive ...
> - Evidence to settle it: Detailed interface statistics (errors, drops, utilization) and routing table stability from both routers ...

Sample size is 2 runs for S2 with the wording on, 1 correct. That is not enough to claim reliability.

## Why the organizers' grader scores that answer 7/10
Two separate effects, both measured:

1. **The grader reads a truncated view.** `score-run.py` is given the audit store's `/timeline` output. That view cuts every record's text at
   exactly 200 characters (longest line in the saved file: 200; the same records in `/records` are 686, 484 and 818 characters). The words
   "reversible" and "Evidence to settle it" come after the cut, so a check that searches for them cannot find them.
2. **The remaining check is a vocabulary list.** `UNCERTAIN_WORDS` = `ambiguous | cannot distinguish | either | inconclusive | insufficient
   evidence | unclear | two possible/plausible | would disambiguate | need(s) more`. Our answer names the evidence but uses none of these words.

Run through the organizers' own CLI on the full records instead of the timeline, the same run scores 8/10 ("proposed something reversible"
flips to PASS; "named what would disambiguate" still fails on vocabulary). We do **not** tune wording to the list; the answer is judged by
reading it.

## How to use the switch (and when not to)
- It is a test harness: an agent that behaves differently depending on a flag we set per scenario is not a production fix. Use it to
  demonstrate that the S2 failure is fixable and what the fix costs. Do not present the agent as handling S2 in general.
- Every assessed C7 and C16 run is made with the switch **off**.
- Each flip changes a governed object (the AgentConfig), so each needs a fresh governance baseline before the run's preflight.

```bash
export AWS_PROFILE=hackathon KUBECONFIG=~/hackathon.kube
cd ~/projects/tools/tmforum-hackathon
# on
kubectl patch agentconfig haiec-network-agent -n components --type merge \
  -p '{"spec":{"awsAgentCore":{"environmentVariables":{"AGENT_AMBIGUITY_PROMPT":"1"}}}}'
sleep 45
.venv/bin/python -m assurance.governance snapshot --out realrun-out/governance/sw-on.json
NETWORK_AGENT=haiec-network-agent BASELINE=realrun-out/governance/sw-on.json \
HAIEC_FROZEN="16=2026-10-05T14:46:12Z,7=2026-10-04T22:56:08Z" HAIEC_LIMITS="16=40000,7=1.0" \
  bash scripts/aws-studio/run-with-agent.sh S2-transport-congestion
# off again (removes the variable)
kubectl patch agentconfig haiec-network-agent -n components --type merge \
  -p '{"spec":{"awsAgentCore":{"environmentVariables":{"AGENT_AMBIGUITY_PROMPT":null}}}}'
```
A rebuild with `new-agent.sh` also resets the variable (it is not part of the generated AgentConfig), which returns the agent to off; it also
resets `STEP_BASE`, so set `STEP_BASE=6` and `IT_AGENT_RUNTIME_ARN` again afterwards.

## Reproduce the evidence in the grader section
```bash
export AWS_PROFILE=hackathon KUBECONFIG=~/hackathon.kube
cd ~/projects/tools/tmforum-hackathon
CID=fault-1791228157-27663b            # the S2 run with the correct answer
SC=S2-transport-congestion
STORE=/api/v1/namespaces/components/services/audit-store:8080/proxy
cp scripts/aws-studio/data/network-inventory.json /tmp/network-inventory.json   # the grader reads this path when given a file

# 1. score it the way the organizers do (timeline view)
kubectl get --raw "$STORE/timeline?cid=$CID" > /tmp/timeline-$CID.txt
python3 scripts/aws-studio/score-run.py $CID $SC /tmp/timeline-$CID.txt | grep -E "gather|disambiguate|reversible|RESULT"

# 2. read the answer in full (the /records endpoint is not truncated)
kubectl get --raw "$STORE/records?cid=$CID" > /tmp/records-$CID.jsonl
python3 -c "
import json
for l in open('/tmp/records-$CID.jsonl'):
    r = json.loads(l)
    if r['actor'] == 'haiec-network-agent' and r['phase'] == 'result-inspection': print(r['detail'].replace('\\\\n', '\n'))"

# 3. the grader's own keyword lists
sed -n 35,39p scripts/aws-studio/score-run.py

# 4. measure the truncation
python3 -c "print('longest line in the timeline view:', max(len(l.strip()) for l in open('/tmp/timeline-$CID.txt')))"
python3 -c "
import json
for l in open('/tmp/records-$CID.jsonl'):
    r = json.loads(l)
    if r['phase'] == 'result-inspection': print('full text length', r['actor'], len(r['detail']))"

# 5. the organizers' own CLI on the full records (it accepts a JSON array)
python3 -c "import json; print(json.dumps([json.loads(l) for l in open('/tmp/records-$CID.jsonl')]))" > /tmp/full-$CID.json
python3 scripts/aws-studio/score-run.py $CID $SC /tmp/full-$CID.json | grep -E "gather|disambiguate|reversible|RESULT"
```
Expected: step 1 gives 7/10 with FAIL on both "named what would disambiguate" and "proposed something reversible"; step 4 prints 200 and
the three full lengths; step 5 gives 8/10 with only "named what would disambiguate" failing (plus the guardrail-string check, see
`docs/FINDINGS.md` O-04, which fails on every run).

## Limits of this evidence
- Two S2 runs with the wording on; one correct. Run-to-run variance is large (S3 without a tool budget spent 40,896 to 45,830 tokens).
- The first table compares runs from consecutive builds (the first hop and IT agent were unchanged); the same-build pair below repeats the
  comparison on one build. Five runs in that pair: one S1 and one S2 off, two S2 and one S1 on. That is a small sample.
- The IT agent's "agree" is a constant string in the organizers' image, so the negotiation records show a real call but a canned reply.

## Same-build off/on comparison
One build of the network agent (image `...3093759def`, switch build), the variable set on the AgentConfig between the runs, a fresh
governance baseline before each group (`sw2-off`, `sw2-on`), preflight GO on all five. Tokens are per agent from the ledger's
`model-request-result` records (equal to the gateway's counts in every run: the auditor's 6 integrity checks passed). The first hop
(`haiec-agent`, 10.6k to 11.2k) and the IT agent (11.7k to 12.0k) did not change between rows.

| Switch | Scenario | Run | Network label | Network tokens | Run total | Largest network input | Tool-budget cancels | Timeline grade | Full-text grade |
|---|---|---|---|---|---|---|---|---|---|
| off | S1 | `fault-1791229989-b23ef8` | auto-resolve | 11,887 | 34,438 | 4,011 | 0 | 7/8 | not computed |
| off | S2 | `fault-1791230092-d4eaa5` | auto-resolve | 16,135 | 39,259 | 4,131 | 0 | 6/10 | 7/10 |
| on | S2 | `fault-1791230246-a341e7` | **gather-evidence** | 20,713 | 43,369 | 4,277 | 1 | 7/10 | 8/10 |
| on | S2 | `fault-1791230347-3bbc6d` | **gather-evidence** | 37,653 | 60,867 | 12,746 | 1 | 7/10 | 8/10 |
| on | S1 | `fault-1791230474-c44bfd` | undetermined | 37,548 | 59,981 | 12,710 | 1 | 7/8 | not computed |

What the pair shows:
- **S2 with the switch on: 2 of 2 correct** (with the always-on build earlier: 3 of 4 in total). Both answers say the root cause is
  undetermined, list the possible causes (transport congestion plus routing asymmetry / CPU and memory exhaustion / interface errors), propose
  a read-only step labelled reversible, and name the evidence that would settle it. With the switch off the answer states one cause
  ("Transport network congestion ... causing intermittent latency SLO breach") and proposes traffic shaping or rerouting.
- **Cost:** network tokens 11.9k and 16.1k with it off, 20.7k and 37.7k with it on. Run totals 34.4k and 39.3k off, 43.4k, 60.9k and 60.0k on:
  all three "on" runs are over the 40,000 cap on our HAIEC system, and two are at the 60,000 mark of the teammate's system (60,867 over,
  59,981 under by 19 tokens). In two of the three "on" runs the network agent had one call with 12.7k input (a large tool reply re-sent
  on every following call) against about 4.1k to 4.3k otherwise; in the run that did not, the cost was smaller.
- **S1 with the switch on:** the network label was `undetermined` because the proposed step became "perform hardware diagnostics ...
  reconfigure the link if needed" (no action word the classifier counts). The switch-off S1 answer proposes "inspect and repair fronthaul
  fiber/optics". Neither names the failing optic (CRC errors on eth-1/7), so this is a difference in tone from one run each, not a
  demonstrated loss of correctness. The organizers' grade is 7/8 either way, because the IT agent's `auto-resolve` satisfies the
  "disposition" check.
- **The "reversible" check is a word match.** On full text the switch-off S2 answer (one confident cause, "validate changes in a maintenance
  window with rollback plan") also passes "proposed something reversible", because `roll ?back` is in the list. The check cannot tell a
  genuinely reversible step from a mention of the word.

State after the job: variable removed, agent Approved, `STEP_BASE=6`, switch off. Every assessed C7 and C16 run is made in that state.

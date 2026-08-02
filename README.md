# Pipeline Deployment Multi-Agent Review (Sample / Demo)

**Day 7** of the "Agentic AI in Data Engineering" series — and the
payoff of a question Day 1
([pipeline-incident-agent](https://github.com/NANDAN-CREATOR/pipeline-incident-agent))
raised and deliberately argued *against* using: multi-agent
architecture. Day 1 explained why one agent with several tools beat
splitting an investigation across multiple agents *for that specific
problem*. This repo is the case where multi-agent genuinely is the
right call — and it shows what changes when it is.

Before a significant pipeline change goes to production, it needs two
genuinely different kinds of review: **risk/reliability** (is this
sized correctly, does it have a rollback plan, is it monitored?) and
**compliance** (does it handle sensitive data correctly, does it
respect residency and retention rules?). This repo runs these as **two
fully independent agents** — different tools, different system prompts,
neither one able to see the other's reasoning — and combines their
verdicts with a **deterministic, non-negotiable reconciliation rule**.

It runs entirely on a **local model via [Ollama](https://ollama.com)** —
no cloud API key required.

> **This is a sample, not a production system.** The "production systems"
> each sub-agent investigates (deployment metadata, resource
> configuration, rollback/monitoring state, a data-handling and
> residency catalog, past incident/compliance-finding history) are
> replaced with small mock backends returning fixed, hand-crafted data
> across four illustrative scenarios. See
> [Adapting This to Real Systems](#adapting-this-to-real-systems) for
> what pointing this at a real environment would actually take.

---

## Why Multi-Agent, Here, When Day 1 Argued Against It

Day 1's single-agent design worked because the whole task was one
continuous chain of reasoning, where each step depended on what the
previous step found. This task is different: risk assessment and
compliance assessment genuinely don't depend on each other, require
different vocabulary and different tools, and cramming both into one
system prompt risks the model blurring the two domains together (e.g.
"approving" on reliability grounds while quietly overlooking a
compliance gap it wasn't really equipped to evaluate). That's exactly
the condition under which Day 1 said multi-agent earns its complexity.

But splitting into two agents creates a brand-new problem neither prior
agent in this series had to solve: **what happens when the two verdicts
don't match?** This repo's central idea is that **reconciling
independent verdicts is itself a decision that must follow a hard,
deterministic rule — not another judgment call handed to a third agent.**
Letting an LLM "resolve" a disagreement between two specialist reviewers
would quietly defeat the entire purpose of having two independent
reviewers in the first place. So the coordinator in `agent.py` is
**plain, deterministic Python — not an LLM call.**

The rule it enforces mirrors the "four-eyes" / maker-checker principle
used in finance and security: **only a clean, unconditional approval
from *both* reviewers can auto-clear.** Any rejection, any conditional
approval, or any mismatch between the two — even two different,
individually reasonable-sounding sets of conditions — routes to a human.

---

## What It Demonstrates

Four independent, runnable scenarios, exercising every meaningful
combination the coordinator has to handle:

| Scenario | Risk verdict | Compliance verdict | Coordinator outcome |
|---|---|---|---|
| `both_approve_clean` | approve | approve | `approve_deployment` — the only case that auto-clears |
| `risk_agent_blocks` | reject (no rollback plan, no load testing, repeat of a past incident's exact gap) | approve | `request_changes` |
| `compliance_agent_blocks` | approve | reject (EU data sent to a non-EU destination, violating residency policy, with direct precedent) | `request_changes` |
| `conflicting_conditional_verdicts_escalate` | approve_with_conditions | approve_with_conditions | `escalate_to_human` — **even though both "approved"** |

The fourth scenario is the one worth sitting with: both reviewers are
individually fine with the deployment, provided their own specific
conditions are met first. It would be easy for a system to treat two
"conditional approvals" as good enough to proceed. This repo's
coordinator explicitly does not — reconciling two different domains'
conditions is a human call, not an automatic merge.

The full coordinator logic was verified directly against every
meaningful combination of the two verdicts (see testing notes in
`agent.py`'s docstring) — all eight combinations produce the expected
outcome, including the case where one sub-agent fails to respond at all.

---

## Requirements

- Python 3.9+
- [Ollama](https://ollama.com) installed and running locally
- A tool-calling-capable model pulled in Ollama, for example:

  ```bash
  ollama pull llama3.1
  ```

  Other models known to support tool calling in Ollama at the time of
  writing include `qwen2.5`, `mistral-nemo`, and `firefunction-v2` — check
  the [Ollama model library](https://ollama.com/library) for current
  tool-calling support. Since this demo runs the model twice per
  scenario (once per sub-agent), it's also a good test of consistency —
  watch whether the same model reliably distinguishes "approve" from
  "approve_with_conditions" when a specific, nameable gap exists, rather
  than defaulting to a clean approval whenever the underlying business
  purpose sounds reasonable.

---

## Setup

```bash
git clone https://github.com/NANDAN-CREATOR/pipeline-deployment-multi-agent-review.git
cd pipeline-deployment-multi-agent-review
pip install -r requirements.txt

# In a separate terminal:
ollama serve
ollama pull llama3.1
```

## Running the Demo

```bash
python agent.py --scenario both_approve_clean
python agent.py --scenario risk_agent_blocks
python agent.py --scenario compliance_agent_blocks
python agent.py --scenario conflicting_conditional_verdicts_escalate
```

Or with a different model:

```bash
python agent.py --scenario conflicting_conditional_verdicts_escalate --model qwen2.5
```

You'll see two separate, clearly-labeled traces — one for the risk
reviewer, one for the compliance reviewer — followed by the
deterministic coordinator's final combined outcome.

**Note on output:** the exact steps and reasoning can vary between runs
and models, since two real model calls are being made per scenario. What
should stay consistent is each sub-agent's *verdict category* for a
given scenario, and — critically — that the coordinator's final outcome
always follows the fixed rule table above regardless of what the two
verdicts happen to be, since that logic is deterministic Python, not
something either model call can influence.

---

## Project Structure

```
pipeline-deployment-multi-agent-review/
├── agent.py           # two independent sub-agents, deterministic coordinator, four scenarios
├── requirements.txt
├── LICENSE
└── README.md
```

Kept as a single file for a sample project like this — see
[Adapting This to Real Systems](#adapting-this-to-real-systems) for how
you'd split it up for a real deployment.

---

## How the Guardrails Work

- **Genuine separation of concerns.** The risk agent and compliance
  agent have entirely separate tool sets and system prompts, and neither
  one's conversation history includes the other's reasoning or verdict.
- **Deterministic reconciliation, not a third agent's judgment call.**
  The `coordinate()` function is plain Python with a fixed rule table —
  no model call is involved in combining the two verdicts, specifically
  so that the reconciliation step can't be "reasoned around" the way an
  LLM call might be.
- **Four-eyes principle.** Only a clean, unconditional double-approval
  auto-clears. Every other combination — including two independent
  conditional approvals — requires a human decision.
- **Sub-agent failure is treated as a blocking condition, not a pass.**
  If either sub-agent fails to reach a verdict within its step budget,
  the coordinator escalates rather than defaulting to approval.
- **Prompt-injection awareness, in both sub-agents.** Each system prompt
  instructs its model to treat tool results as data to evaluate, never
  as commands to follow.
- **Step limit per sub-agent.** A hard cap (`MAX_STEPS`, default 12 per
  sub-agent) forces that sub-agent's run to end without a verdict rather
  than loop indefinitely — which the coordinator then treats as an
  escalation condition.

---

## Adapting This to Real Systems

Only the **bodies** of these nine read-only functions in `agent.py` need
to change to point this at a real environment:

| Function | Would call, in a real deployment |
|---|---|
| `get_deployment_details` | Your CI/CD or deployment-management tool |
| `get_resource_configuration` | Your cluster manager / infrastructure-as-code config |
| `get_error_handling_and_rollback_plan` | A deployment runbook or your deployment tool's rollback configuration |
| `get_monitoring_coverage` | Your observability platform's alert/monitor registry |
| `search_past_deployment_incidents` | Your incident tracker, filtered to deployment-related causes |
| `get_data_handling_summary` | Your data catalog's classification tagging (the same kind of system Day 5's agent relies on) |
| `get_data_residency_requirements` | Your documented data residency policy per data category |
| `get_retention_policy_compliance` | Your documented retention policy and the deployment's actual configuration |
| `search_past_compliance_findings` | Your compliance/governance review history |

The tool **schemas**, the **system prompts**, the **sub-agent loop**,
and — especially — the **coordinator's rule table** don't need to
change. You'd also want to, at minimum:

- Get whoever owns your deployment approval process to explicitly sign
  off on the reconciliation rule table itself (what combinations
  auto-clear vs. escalate) — this is a governance decision, not
  something to leave implicit in code
- Route `request_changes` and `escalate_to_human` outcomes to wherever
  your team actually reviews deployment approvals, with both verdicts
  and their full rationale attached
- Consider adding a third, independent sub-agent for another domain
  (e.g. a cost-impact reviewer, echoing Day 4) if a third genuinely
  distinct area of expertise is needed — the coordinator's rule table
  would simply need to account for a third verdict, still deterministically
- Build a scenario-based test suite exactly like this repo's four
  scenarios, but drawn from your own team's real past deployment
  reviews, and verify the coordinator's outcome table continues to match
  your organization's actual approval policy as it evolves

---

## License

MIT — see [LICENSE](LICENSE).

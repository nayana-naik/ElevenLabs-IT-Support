# Architecture

## Call flow

```
Caller
  |
  v
Agent 1 - Verification
  - greets caller
  - calls tool verify_employee -> n8n -> (fake employee list)
  - not verified (after 1 retry) --------------------> Agent 4 - Escalation
  - asks "billing or technical issue?"
  |
  |-- billing --> Agent 2 - Billing
  |                 - tries to resolve directly
  |                 - unresolved -> create_ticket -> transfer_to_escalation --> Agent 4
  |
  '-- technical --> Agent 3 - Tech Support
                      - 1-2 troubleshooting steps
                      - unresolved -> create_ticket -> transfer_to_escalation --> Agent 4

Agent 4 - Escalation
  - calls escalate_to_human -> n8n -> escalation_id
  - reads reference number back to caller, ends call

[call ends]
  -> ElevenLabs sends post-call webhook -> n8n "Post-Call Structured Review" workflow
  -> builds a structured review: resolved? process followed? follow-up needed?
```

## Why 2 n8n workflows instead of a backend server

- `n8n/it-support-tools.workflow.json` - 3 webhook endpoints the agents call *during* the
  call: `verify-employee`, `create-ticket`, `escalate`. All logic is a few lines of
  JS in a Code node, no database - data is a hardcoded fake array (see `data/employees.sample.json`).
- `n8n/post-call-review.workflow.json` - 1 webhook ElevenLabs calls *after* the call ends,
  which reshapes ElevenLabs' built-in evaluation + data-collection results into one clean
  JSON object (the "structured review").

This keeps the whole project to "webhook in -> small JS function -> respond" with zero
servers to host, zero databases, and zero extra API calls (no extra LLM calls either -
the structured review is built from data ElevenLabs already generates for free as part
of the call).

## Why agent-to-agent transfer instead of one agent with branching logic

ElevenLabs Conversational AI has a native `transfer_to_agent` tool. Using 4 separate
agents (verification / billing / tech support / escalation) instead of one mega-agent
with if/else prompts makes each agent's job small, easy to explain in a course writeup,
and easy to demo one piece at a time.

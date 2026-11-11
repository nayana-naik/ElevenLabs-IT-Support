# Agents setup

**Recommended path: use the API (curl), not the dashboard template picker.**
Creating an agent in the dashboard from any template other than **Blank**
(e.g. "Customer Service") attaches a hidden Workflow graph that can make the
agent end calls after one turn, or reference tools that don't exist. The curl
flow below never hits that trap. Full copy-pasteable commands:
[`docs/API_SETUP_GUIDE.md`](../docs/API_SETUP_GUIDE.md).

## Templates vs. real files (keeping IDs confidential)

- `agents/templates/*.json` - clean placeholder versions (`{{AGENT_2_BILLING_ID}}`,
  `{{VERIFY_EMPLOYEE_TOOL_ID}}`, etc). These are safe for GitHub.
- `agents/1-verification-agent.json`, `2-billing-agent.json`, `3-tech-support-agent.json`,
  `4-escalation-agent.json`, `tools.json` - your local working copies, filled in
  with real ElevenLabs IDs and your n8n URL. These (and `agentids.txt`) are listed
  in `.gitignore` so they never get committed.

## What each file actually is

These are **request bodies**, not a full export of an agent - they're what you
`POST`/send to the API, not everything ElevenLabs stores:

- `tools.json` - 3 webhook tools (`verify_employee`, `create_ticket`,
  `escalate_to_human`). Tools are standalone objects in ElevenLabs
  (`POST /v1/convai/tools`) - an agent references them by `tool_id`, it doesn't
  embed them.
- `1-verification-agent.json` ... `4-escalation-agent.json` - one
  `POST /v1/convai/agents/create` body each. Routing between agents uses the
  built-in `transfer_to_agent` system tool (`prompt.built_in_tools.transfer_to_agent`),
  not a separate tool per transfer.

## Setup order

1. Create the 3 tools from `tools.json` first - note each returned `tool_id`.
2. Create the 4 agents in this order, since each later one needs an earlier
   one's `agent_id` for its transfer rules: **Escalation → Billing → Tech
   Support → Verification**.

Full commands for both steps: `docs/API_SETUP_GUIDE.md`.

## Placeholders to replace (in `templates/`)

| Placeholder | Replace with |
|---|---|
| `{{N8N_WEBHOOK_BASE_URL}}` | your n8n webhook base URL (see `n8n/README.md`) |
| `{{VERIFY_EMPLOYEE_TOOL_ID}}`, `{{CREATE_TICKET_TOOL_ID}}`, `{{ESCALATE_TOOL_ID}}` | `tool_id`s returned when you create the tools |
| `{{AGENT_2_BILLING_ID}}`, `{{AGENT_3_TECH_SUPPORT_ID}}`, `{{AGENT_4_ESCALATION_ID}}` | `agent_id`s returned when you create those agents |

## Saving ElevenLabs credits (free plan, ~10K credits)

- Voice model is `eleven_turbo_v2` (English agents reject the `_v2_5` suffix -
  cheapest, lowest-latency option that's actually accepted).
- Keep `first_message` and prompts short - shorter agent speech = fewer characters billed.
- Test the *tool logic* via n8n directly first (curl the 3 webhooks, see
  `n8n/README.md`) before doing real voice test calls - that costs 0 ElevenLabs credits.
- Do a small number of full voice test calls at the end, not during development.

## Known gap: structured review isn't wired up yet

`docs/POST_CALL_REVIEW.md` describes evaluation criteria, data collection, and
a post-call webhook - none of that is set on the live agents yet (it requires
creating a separate workspace-level webhook object and attaching its ID, see
`docs/API_SETUP_GUIDE.md`'s notes). The 4 agents work end-to-end without it;
this only affects whether the `post-call-review` n8n workflow ever gets called.

## Agent responsibilities

1. **Verification** - verifies caller identity, routes to Billing or Tech Support
2. **Billing** - billing/account IT issues
3. **Tech Support** - technical IT issues
4. **Escalation** - hands off to a human, ends the call

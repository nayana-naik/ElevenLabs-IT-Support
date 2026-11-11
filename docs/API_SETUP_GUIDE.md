# Setting up the 4 agents via the ElevenLabs API (curl)

This is a record of every `curl` call used to provision this project's tools and
agents directly against the ElevenLabs REST API, bypassing the dashboard UI
entirely. Useful if you can't debug through the browser console, want to
re-create the agents from scratch, or are scripting this project's setup.

All commands assume:
```bash
API_KEY="<your ElevenLabs API key>"          # xi-api-key header
N8N_WEBHOOK_BASE_URL="<your n8n base URL>"   # e.g. https://your-tunnel.example/webhook
```

## Lessons learned (read this before copying commands)

- **Don't send both `tools` and `tool_ids`.** A `GET` on an agent echoes a
  read-only expanded `tools` array (resolved from `tool_ids` + system tools). If
  you round-trip that same JSON back into a `PATCH`/`POST`, the API rejects it
  with `both_tools_and_tool_ids_provided`. Always clear
  `conversation_config.agent.prompt.tools` to `[]` before sending a body that
  also sets `tool_ids`.
- **Don't attach a `workflow` graph to a flat agent.** If you `POST
  /v1/convai/agents/create` **without** a `workflow` key, you get a classic
  flat agent driven purely by `conversation_config.agent.prompt`. If a
  `workflow` object exists (e.g. because the agent was created from a
  dashboard template like "Customer Service"), the agent's conversation is
  driven by graph nodes instead - a node's outgoing edge can auto-fire after
  a single turn (`exit_slot: "any"`) and end the call prematurely, regardless
  of what's in the top-level prompt. There's no supported way found here to
  clear an existing `workflow` via `PATCH` (`workflow: null` is silently
  ignored) - the reliable fix is deleting and recreating the agent without a
  `workflow` field.
- **English agents need `eleven_turbo_v2` or `eleven_flash_v2`**, not
  `_v2_5` variants - `POST`ing `eleven_turbo_v2_5` as the TTS `model_id` fails
  validation with `"English Agents must use turbo or flash v2."`
- **For safe partial edits**, `GET` the full agent, modify only the fields you
  need in the full JSON object client-side, then `PATCH` the complete object
  back - don't assume nested objects are deep-merged by the API.

## 1. Sanity checks

```bash
# Confirm the key works (expect a permissions-scoped response, not a network error)
curl -s https://api.elevenlabs.io/v1/user -H "xi-api-key: $API_KEY"

# List tools already in the account
curl -s https://api.elevenlabs.io/v1/convai/tools -H "xi-api-key: $API_KEY"

# List agents already in the account
curl -s https://api.elevenlabs.io/v1/convai/agents -H "xi-api-key: $API_KEY"
```

## 2. Create the 3 webhook tools

```bash
# verify_employee
curl -s -X POST https://api.elevenlabs.io/v1/convai/tools \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "tool_config": {
      "type": "webhook",
      "name": "verify_employee",
      "description": "Look up an employee by employee_id or email to confirm they are a real employee before discussing their account.",
      "response_timeout_secs": 20,
      "api_schema": {
        "url": "'"$N8N_WEBHOOK_BASE_URL"'/verify-employee",
        "method": "POST",
        "request_body_schema": {
          "type": "object",
          "properties": {
            "employee_id": {"type": "string", "description": "Employee ID, e.g. E1001"},
            "email": {"type": "string", "description": "Work email, used if employee_id is not given"}
          }
        }
      }
    }
  }'

# create_ticket
curl -s -X POST https://api.elevenlabs.io/v1/convai/tools \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "tool_config": {
      "type": "webhook",
      "name": "create_ticket",
      "description": "Log an IT billing or technical issue that needs a real change made, or a human to finish resolving.",
      "response_timeout_secs": 20,
      "api_schema": {
        "url": "'"$N8N_WEBHOOK_BASE_URL"'/create-ticket",
        "method": "POST",
        "request_body_schema": {
          "type": "object",
          "properties": {
            "employee_id": {"type": "string", "description": "Employee ID confirmed via verify_employee"},
            "category": {"type": "string", "description": "billing or technical"},
            "issue_summary": {"type": "string", "description": "Short summary of the issue"}
          }
        }
      }
    }
  }'

# escalate_to_human
curl -s -X POST https://api.elevenlabs.io/v1/convai/tools \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "tool_config": {
      "type": "webhook",
      "name": "escalate_to_human",
      "description": "Hand the call off to a live human IT support agent and get back a reference number.",
      "response_timeout_secs": 20,
      "api_schema": {
        "url": "'"$N8N_WEBHOOK_BASE_URL"'/escalate",
        "method": "POST",
        "request_body_schema": {
          "type": "object",
          "properties": {
            "reason": {"type": "string", "description": "Why the call is being escalated"},
            "ticket_id": {"type": "string", "description": "ticket_id from create_ticket, if one exists"}
          }
        }
      }
    }
  }'
```

Each response includes an `"id": "tool_xxxxx"` - save these three IDs, they're
needed in the next step.

To fix an existing tool instead of creating a new one (e.g. wrong HTTP method):

```bash
curl -s -X PATCH https://api.elevenlabs.io/v1/convai/tools/$TOOL_ID \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"tool_config": {"type":"webhook","name":"escalate_to_human","api_schema":{"url":"'"$N8N_WEBHOOK_BASE_URL"'/escalate","method":"POST","request_body_schema":{...}}}}'
```

## 3. Create the 4 agents (no `workflow` field)

Create in this order, since each later agent needs an earlier agent's ID for
its transfer rules: **Escalation → Billing → Tech Support → Verification**.

```bash
# Escalation (no transfers - end of the line)
curl -s -X POST https://api.elevenlabs.io/v1/convai/agents/create \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Escalation-Agent",
    "conversation_config": {
      "agent": {
        "first_message": "I understand - let me get you connected to a live IT support person right now.",
        "language": "en",
        "prompt": {
          "prompt": "You are the Escalation agent ...",
          "llm": "gemini-3.1-flash-lite",
          "temperature": 0.4,
          "tool_ids": ["'"$ESCALATE_TOOL_ID"'"]
        }
      },
      "tts": {"model_id": "eleven_turbo_v2", "voice_id": "21m00Tcm4TlvDq8ikWAM"}
    }
  }'
# -> save the returned agent_id as $ESCALATION_ID

# Billing (transfers to Escalation)
curl -s -X POST https://api.elevenlabs.io/v1/convai/agents/create \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Billing-Agent",
    "conversation_config": {
      "agent": {
        "first_message": "Thanks, I'\''ve got your billing question - go ahead and tell me what'\''s happening.",
        "language": "en",
        "prompt": {
          "prompt": "You are the Billing & Accounts specialist ...",
          "llm": "gemini-3.1-flash-lite",
          "temperature": 0.4,
          "tool_ids": ["'"$CREATE_TICKET_TOOL_ID"'"],
          "built_in_tools": {
            "transfer_to_agent": {
              "type": "system",
              "name": "transfer_to_agent",
              "description": "Transfer the caller to Escalation if the billing issue can'\''t be resolved or ticketed, or they ask for a human.",
              "params": {
                "system_tool_type": "transfer_to_agent",
                "transfers": [
                  {"agent_id": "'"$ESCALATION_ID"'", "condition": "Billing issue cannot be resolved or ticketed, or caller asks for a human", "delay_ms": 0, "transfer_message": null, "enable_transferred_agent_first_message": true}
                ]
              }
            }
          }
        }
      },
      "tts": {"model_id": "eleven_turbo_v2", "voice_id": "21m00Tcm4TlvDq8ikWAM"}
    }
  }'
# -> save as $BILLING_ID

# Tech Support (transfers to Escalation) - same shape as Billing, different prompt/first_message
# -> save as $TECHSUPPORT_ID

# Verification (transfers to Billing, Tech Support, or Escalation)
curl -s -X POST https://api.elevenlabs.io/v1/convai/agents/create \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Verification-Agent",
    "conversation_config": {
      "agent": {
        "first_message": "Thanks for calling Acme Corp IT Support. Can I get your employee ID or work email to pull up your account?",
        "language": "en",
        "prompt": {
          "prompt": "You are the IT Support intake agent ...",
          "llm": "gemini-3.1-flash-lite",
          "temperature": 0.4,
          "tool_ids": ["'"$VERIFY_EMPLOYEE_TOOL_ID"'"],
          "built_in_tools": {
            "transfer_to_agent": {
              "type": "system",
              "name": "transfer_to_agent",
              "description": "Transfer the caller to the right specialist agent based on their request.",
              "params": {
                "system_tool_type": "transfer_to_agent",
                "transfers": [
                  {"agent_id": "'"$BILLING_ID"'", "condition": "Issue is billing / licensing / subscription / billing-access related", "delay_ms": 0, "transfer_message": null, "enable_transferred_agent_first_message": true},
                  {"agent_id": "'"$TECHSUPPORT_ID"'", "condition": "Issue is a technical / hardware / software / login problem", "delay_ms": 0, "transfer_message": null, "enable_transferred_agent_first_message": true},
                  {"agent_id": "'"$ESCALATION_ID"'", "condition": "Caller cannot be verified after one retry, or explicitly asks for a human", "delay_ms": 0, "transfer_message": null, "enable_transferred_agent_first_message": true}
                ]
              }
            }
          }
        }
      },
      "tts": {"model_id": "eleven_turbo_v2", "voice_id": "21m00Tcm4TlvDq8ikWAM"}
    }
  }'
```

## 4. Inspect / debug an agent or call

```bash
# Full current config for one agent
curl -s "https://api.elevenlabs.io/v1/convai/agents/$AGENT_ID" -H "xi-api-key: $API_KEY"

# Recent conversations for one agent (status, termination_reason, cost)
curl -s "https://api.elevenlabs.io/v1/convai/conversations?agent_id=$AGENT_ID&page_size=5" -H "xi-api-key: $API_KEY"

# Full transcript + tool calls for one conversation
curl -s "https://api.elevenlabs.io/v1/convai/conversations/$CONVERSATION_ID" -H "xi-api-key: $API_KEY"
```

## 5. Update an existing agent safely

```bash
# 1. GET the full current config
curl -s "https://api.elevenlabs.io/v1/convai/agents/$AGENT_ID" -H "xi-api-key: $API_KEY" -o agent.json

# 2. Edit agent.json with a script/editor - change only what you need,
#    and set conversation_config.agent.prompt.tools to [] if prompt.tool_ids is set.

# 3. PATCH the full object back (only top-level fields the API accepts: name,
#    conversation_config, platform_settings, workflow, tags)
curl -s -X PATCH "https://api.elevenlabs.io/v1/convai/agents/$AGENT_ID" \
  -H "xi-api-key: $API_KEY" -H "Content-Type: application/json" \
  --data-binary @agent-patch-body.json
```

## 6. Delete an agent

```bash
curl -s -X DELETE "https://api.elevenlabs.io/v1/convai/agents/$AGENT_ID" -H "xi-api-key: $API_KEY"
```

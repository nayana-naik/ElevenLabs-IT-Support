# AI IT Support Call System (ElevenLabs + n8n)

A voice-based AI support system that takes an employee IT support call from intake
to resolution (or escalation), using 4 specialized ElevenLabs Conversational AI
agents and 3 tiny n8n workflows. After each call, n8n turns ElevenLabs' built-in
call analysis into a structured review (resolved? right process followed?
follow-up needed?).

## Project structure

```
.
├── agents/        4 ElevenLabs agent configs + setup guide   <- EDIT HERE (API key use + agent setup)
│   ├── README.md
│   ├── tools.json                     webhook tool defs (real URLs/IDs - gitignored)
│   ├── 1-verification-agent.json      (real Agent IDs - gitignored)
│   ├── 2-billing-agent.json           (real Agent IDs - gitignored)
│   ├── 3-tech-support-agent.json      (real Agent IDs - gitignored)
│   ├── 4-escalation-agent.json        (real Agent IDs - gitignored)
│   └── templates/                     placeholder versions, safe for GitHub
├── n8n/           3 n8n workflows (the "backend") + setup guide   <- n8n CONFIGURED HERE
│   ├── README.md
│   ├── it-support-tools.workflow.json
│   ├── post-call-review.workflow.json
│   └── error-notifications.workflow.json   optional Slack/Discord alerts on failure
├── twilio/        optional real-phone-number integration
│   ├── README.md
│   └── twilio-inbound-call.workflow.json
├── data/
│   └── employees.sample.json     fake employee directory (also embedded in the n8n workflow)
├── docs/
│   ├── ARCHITECTURE.md           full call flow diagram
│   ├── POST_CALL_REVIEW.md       structured review schema + example
│   └── API_SETUP_GUIDE.md        curl-only setup/debug reference (no dashboard needed)
├── .env.example      <- ELEVENLABS_API_KEY and all other config go here
└── .gitignore
```

## The 4 agents

| # | Agent | Job |
|---|---|---|
| 1 | Verification | Verifies caller identity, asks what's wrong, routes to Billing or Tech Support |
| 2 | Billing | Resolves billing/license/subscription issues, or tickets + escalates |
| 3 | Tech Support | Resolves password/VPN/hardware/software issues, or tickets + escalates |
| 4 | Escalation | Hands off to a live human, gives the caller a reference number, ends call |

## Quick start

1. **ElevenLabs**: create a free account, get your API key -> put it in `.env`
   (copy from `.env.example`). Details: `agents/README.md`.
2. **n8n**: import the 2 workflow files from `n8n/`, activate them, copy the
   webhook URLs into `.env`. Details: `n8n/README.md`.
3. **Agents**: create the 3 tools then the 4 agents via `curl` using the JSON
   bodies in `agents/` (don't use a dashboard template other than Blank - it
   attaches a hidden Workflow graph that breaks this design). Full commands:
   `docs/API_SETUP_GUIDE.md`. Details: `agents/README.md`.
4. **Test**: talk to Agent 1 via the ElevenLabs web widget/test-call feature.
   Try: verify as `E1002` / `ravi.mehta@acmecorp.com`, report a VPN issue, let it
   escalate.
5. **(Optional) Twilio**: see `twilio/README.md` to put this behind a real phone number.

## Why it's structured this way

- **Minimum code**: the entire "backend" is two n8n workflows, each just
  `Webhook -> a few lines of JS -> Respond`. No server to deploy, no database.
- **Minimum ElevenLabs credits**: cheapest TTS model, short prompts, and all tool
  logic testable via `curl` against n8n before spending any voice minutes. See
  `agents/README.md` -> "Saving ElevenLabs credits".
- **Fake data on purpose**: `data/employees.sample.json` has 3 made-up employees -
  enough to demo verification working/failing without any real PII.
- Full flow + rationale: `docs/ARCHITECTURE.md`. Structured review format:
  `docs/POST_CALL_REVIEW.md`.

## Editing with Codex / another machine

Every workflow and agent config is a plain JSON file with inline `{{PLACEHOLDER}}`
values - no project-specific build step, no local server required to edit. You
only need a browser to actually click through ElevenLabs/n8n UIs; all the logic
itself can be written/reviewed as plain text/JSON here first.

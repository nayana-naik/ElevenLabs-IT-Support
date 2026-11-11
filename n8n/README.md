# n8n setup

Three workflows. All are self-contained (no database needed) - "backend" logic is
a few lines of JavaScript in each Code node.

| File | Purpose |
|---|---|
| `it-support-tools.workflow.json` | 3 webhook tools the agents call during a call |
| `post-call-review.workflow.json` | builds the structured review after a call ends |
| `error-notifications.workflow.json` | optional - POSTs error details to a webhook URL you choose |

## 1. Import

In n8n: **Workflows -> Import from File** -> pick `it-support-tools.workflow.json`.
Repeat for `post-call-review.workflow.json` and `error-notifications.workflow.json`.

## 2. Activate

Open each imported workflow and click **Active** (top right). Inactive workflows
won't respond to real webhook calls.

## 2b. (Optional) Wire up error logging

Errors get POSTed as JSON to any endpoint you pick - a webhook.site URL for quick
testing, a Slack/Discord incoming webhook, or your own server.

1. Open `error-notifications.workflow.json`'s "HTTP Request - Send Error" node and
   replace the placeholder URL with your endpoint.
2. On both `it-support-tools` and `post-call-review` workflows: **⋮ menu -> Settings ->
   Error Workflow** -> select "IT Support - Error Notifications".
3. If a tool call or review generation fails, that endpoint receives
   `{timestamp, workflow, node, error, execution_url}` instead of it failing silently.

## 3. Get the webhook URLs

Click each **Webhook** node -> copy its **Production URL**. You'll get:
- `.../webhook/verify-employee`
- `.../webhook/create-ticket`
- `.../webhook/escalate`
- `.../webhook/post-call-review`

The base part (everything before `/verify-employee` etc.) is your
`N8N_WEBHOOK_BASE_URL` - put it in `.env` and in the agent JSON files
(see `agents/README.md`).

## 4. Where n8n needs to be running

- **Local**: `npx n8n` (needs Node.js) - fine for development, but ElevenLabs
  needs a URL it can reach over the internet, so use a tunnel (n8n prints one
  automatically with `npx n8n`) when testing real calls.
- **n8n Cloud**: sign up free at n8n.io, import the same 2 files - URLs are
  public by default, no tunnel needed. Recommended for the actual course demo.

## 5. Test without spending ElevenLabs credits

```bash
curl -X POST "$N8N_WEBHOOK_BASE_URL/verify-employee" \
  -H "Content-Type: application/json" \
  -d '{"employee_id":"E1002"}'

curl -X POST "$N8N_WEBHOOK_BASE_URL/create-ticket" \
  -H "Content-Type: application/json" \
  -d '{"employee_id":"E1002","category":"technical","issue_summary":"VPN wont connect"}'

curl -X POST "$N8N_WEBHOOK_BASE_URL/escalate" \
  -H "Content-Type: application/json" \
  -d '{"reason":"VPN issue not resolved by troubleshooting"}'
```

If all 3 return JSON, the agents' tools will work once the URLs are pasted into
the agent configs.

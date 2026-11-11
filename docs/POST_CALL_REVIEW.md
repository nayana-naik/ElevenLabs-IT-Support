# Post-call structured review

After every call, ElevenLabs sends a "post-call webhook" to n8n with the full
transcript plus two things configured on each agent (`platform_settings` in the
agent JSON files):

- **evaluation.criteria** - 3 yes/no/partial questions ElevenLabs' own model answers
  by reading the transcript: `resolved`, `process_followed`, `follow_up_required`.
- **data_collection** - 3 fields pulled out of the transcript: `employee_id`,
  `issue_category`, `ticket_id`.

The `n8n/post-call-review.workflow.json` workflow just reshapes that payload into one
clean object:

```json
{
  "call_id": "conv_123",
  "agent_id": "agent_456",
  "call_successful": "success",
  "issue_resolved": "success",
  "correct_process_followed": "success",
  "follow_up_required": "failure",
  "employee_id": "E1002",
  "issue_category": "technical",
  "ticket_id": "TCK-482913",
  "summary": "Caller had a VPN connection issue, resolved after reconnect.",
  "generated_at": "2026-11-10T12:00:00.000Z"
}
```

This is the "structured review" the project brief asks for: did it get resolved, was
the right process followed (verify -> route -> resolve/escalate), and is follow-up
needed (open ticket or pending human handoff).

No extra LLM call is made for this - it's free, reusing analysis ElevenLabs already
does. If ElevenLabs' payload field names differ slightly from what's assumed in the
Code node, check the current shape at
https://elevenlabs.io/docs/conversational-ai/workflows/post-call-webhooks and adjust
the `n8n/post-call-review.workflow.json` Code node (it's ~25 lines).

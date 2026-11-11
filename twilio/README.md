# Twilio integration (optional)

Not required to run this project - ElevenLabs agents work with a web call widget
or an ElevenLabs-hosted number out of the box. This is only for routing a *real
phone number* through Twilio.

## Option A (recommended, no n8n needed)

ElevenLabs dashboard -> **Phone Numbers -> Import from Twilio**. Enter:
- `TWILIO_ACCOUNT_SID`
- `TWILIO_AUTH_TOKEN`
- the Twilio phone number's SID

ElevenLabs then owns the inbound call handling directly. Assign **Agent 1
(Verification)** as the number's agent. Done.

## Option B (manual, if you want Twilio/n8n in the middle)

Use `twilio-inbound-call.workflow.json`:
1. Import + activate it in n8n (same steps as `n8n/README.md`).
2. In the Twilio console, open your phone number -> **Voice & Fax -> "A call
   comes in"** -> set the webhook to this workflow's Production URL.
3. Open the workflow's "Code - Build TwiML" node and set `ELEVENLABS_SIP_URI`
   to the SIP URI shown on Agent 1's phone number settings in ElevenLabs.

This makes Twilio forward the call to ElevenLabs over SIP via TwiML's `<Dial><Sip>`,
instead of ElevenLabs managing the number directly.

## Credit note

Phone/Twilio calls still consume ElevenLabs minutes the same as the web widget.
Prefer Option A for the course demo to keep setup minimal; only use Option B if the
assignment specifically asks for a Twilio-in-the-loop architecture.

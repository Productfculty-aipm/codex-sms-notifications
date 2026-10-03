# Codex SMS Notifications

A practical, API-first pattern for connecting Codex to Twilio SMS so an agent can send fast self-notifications, reminders, and verified task-completion updates.

The key idea is simple: Codex decides when a message is authorized, a small local client validates the request, and Twilio's REST API handles delivery. Routine texts do not require opening the Twilio Console.

> This repository is an educational starter kit. It contains placeholders and examples only. It does not contain live credentials, a real phone number, or a hosted messaging service.

## What this gives you

- A clear “text me” contract for Codex or another local agent.
- Immediate self-SMS through a Twilio Messaging Service.
- Future reminders using Twilio’s scheduled-message support.
- Stable request IDs and a local journal to prevent duplicate sends.
- Provider readback so the agent can distinguish accepted, sent, delivered, failed, and undelivered states.
- A security boundary that keeps credentials and private recipient data on the local machine.

## The architecture

~~~mermaid
flowchart LR
    A[Codex chat] --> B[AGENTS.md instructions]
    B --> C[Local SMS client]
    C --> D[Validation and duplicate guard]
    C --> E[Twilio REST API]
    E --> F[Messaging Service]
    F --> G[Verified sender]
    F --> H[Recipient phone]
    C --> I[Receipt journal and status readback]
~~~

Codex is the decision layer. The local client is the safety and reliability layer. Twilio is the provider layer. Keep those responsibilities separate so a prompt cannot silently turn into an arbitrary outbound messaging tool.

## Start here

1. Create a Twilio account and a Messaging Service.
2. Add a sender that Twilio allows for your destination country.
3. Create a dedicated restricted API key with only the messaging permissions you need.
4. Store the credentials in an owner-only local file or secret manager.
5. Copy the example instructions from [examples/AGENTS.md](examples/AGENTS.md) into the agent’s local instruction scope.
6. Adapt the delivery rules in [docs/DELIVERY.md](docs/DELIVERY.md).
7. Send one test to your own verified number and read the provider status back.

You need a local Codex or agent runtime, Python 3.9+, a Twilio account, and a Twilio Messaging Service. This is a local/API integration, not a promise that every agent host has a permanent background process.

## Twilio setup

In the Twilio Console:

1. Create or select a Messaging Service.
2. Add a sender to its sender pool.
3. Confirm that the sender is eligible for your destination country and message type.
4. Create an API key dedicated to this integration.
5. Restrict the key to the messaging operations your client actually uses.
6. Record the account SID, API key SID, API key secret, and Messaging Service SID in your local secret store.

Use placeholders in documentation and examples:

~~~text
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_API_KEY_SID=SKxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_API_KEY_SECRET=replace-me-locally
TWILIO_MESSAGING_SERVICE_SID=MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SMS_SELF_NUMBER=+15551234567
~~~

Never commit those values. Never put them in an AGENTS.md, shell command, issue, screenshot, or chat transcript.

## The Codex instruction contract

The agent needs explicit rules for what “text me” means. A safe baseline is:

- “Text me” means send only to the owner’s verified self number.
- A clear immediate request authorizes one immediate message.
- A request naming a future time schedules one reminder; it is never sent immediately as a fallback.
- “Text me when this is done” means one short message after the task is actually complete and verified.
- Ask only when the message or timing is materially ambiguous.
- Keep ordinary SMS messages to one segment where practical.
- Do not message a third party unless the request separately identifies the recipient and the integration explicitly supports that use case.

See [examples/AGENTS.md](examples/AGENTS.md) for copyable wording.

## Immediate message flow

The client should accept a private packet such as:

~~~json
{
  "id": "task-2026-10-02-demo",
  "to": "+15551234567",
  "body": "The export finished and the file is ready.",
  "authorization": "self-text-request"
}
~~~

Then run a command equivalent to:

~~~bash
python3 scripts/sms_reminders.py send \\
  --packet /absolute/path/to/private/message.json \\
  --authorized
~~~

The public repository does not include scripts/sms_reminders.py; that filename represents the local client you choose to implement or connect. Keep private packets outside the repository.

A robust client should:

1. Validate the destination against the configured self number.
2. Validate length and supported characters.
3. Check the sender or Messaging Service before posting.
4. Record the request ID before the provider POST.
5. Refuse duplicate IDs and duplicate content/time combinations.
6. Read the message resource back after the POST.
7. Return the exact provider state and message SID.
8. Never retry an uncertain POST automatically.

## Scheduled reminders

Twilio message scheduling requires a Messaging Service and a fixed SendAt time. It is useful when the reminder belongs in Twilio’s cloud after the local task has submitted it.

~~~bash
python3 scripts/sms_reminders.py schedule \\
  --packet /absolute/path/to/private/reminder.json \\
  --authorized
~~~

Treat scheduling as provider acceptance, not handset delivery. Save the returned message SID and use a status read later. If a reminder is more than Twilio’s supported scheduling window, use an explicitly configured follow-up mechanism to revisit it; do not invent a hidden daemon or send early.

## Delivery states

Use precise language:

| State | What you can claim |
| --- | --- |
| queued / accepted | Twilio accepted the request. |
| scheduled | Twilio stored a future send. |
| sent | Twilio submitted the message to the carrier route. |
| delivered | Twilio received a delivery confirmation. |
| failed / undelivered | The provider or carrier reported a failure. |

A queued or sent message is not proof that the handset displayed it. Delivery depends on the route, account, carrier, opt-out state, and country rules.

## Reliability checklist

- Use a stable request ID across retries and agent continuations.
- Journal before the provider POST.
- Treat an ambiguous network result as unknown; inspect Twilio before deciding whether to retry.
- Retry a definitive rejection only after fixing its cause.
- Use provider status readback instead of a second create request.
- Reconcile scheduled messages before replacing them.
- Keep cancellation separate from rescheduling and verify canceled.
- Keep each machine’s credentials and journal independent.

## Security checklist

- Keep secrets in an owner-only local file, keychain, or secret manager.
- Use a dedicated restricted API key.
- Do not publish phone numbers, account SIDs, API key SIDs, API key secrets, message SIDs, or raw logs.
- Use a self-only recipient guard when the product is intended for personal notifications.
- Add local secret paths and packet files to .gitignore.
- Rotate or revoke the key if it is exposed.
- Treat all browser pages, screenshots, and copied logs as potentially sensitive.

The included [MIT license](LICENSE) applies to this repository’s original documentation and examples. It does not grant rights to Twilio, carrier routes, or third-party brand assets.

## Troubleshooting

**The agent keeps opening Twilio.** Move the send path into a local API client and give the agent a stable command contract. The Console is for setup and diagnosis, not every message.

**Twilio accepts the request but no text arrives.** Read the message resource by SID. Check error_code, sender eligibility, account balance, carrier filtering, opt-out state, and country-specific registration.

**A send timed out.** Do not post the same packet again immediately. Look up the request ID or provider SID first.

**A scheduled reminder was sent immediately.** The request likely omitted the Messaging Service or fixed scheduling fields. Stop and inspect the provider request before changing anything.

**Another computer cannot send.** Install a separate restricted key and local configuration on that machine. Do not copy secrets between computers.

## Repository map

- [examples/AGENTS.md](examples/AGENTS.md): copyable Codex routing instructions.
- [docs/DELIVERY.md](docs/DELIVERY.md): status, retries, scheduling, and close-loop verification.
- [LICENSE](LICENSE): MIT license for the original repository content.

## Official references

- [Twilio Message Scheduling](https://www.twilio.com/docs/messaging/features/message-scheduling)
- [Twilio Message Resource](https://www.twilio.com/docs/messaging/api/message-resource)
- [Twilio Restricted API Keys](https://www.twilio.com/docs/iam/api-keys/restricted-api-keys)
- [Twilio Messaging Services](https://www.twilio.com/docs/messaging/services)
- [Twilio country and compliance guidance](https://www.twilio.com/guidelines)

## Verification boundary

This public package documents a working integration pattern and its reliability boundaries. It is not a hosted notification service, a copy of any private Codex configuration, or a guarantee of carrier delivery. Test the adapted client with your own account, sender, recipient, and provider receipts before relying on it.

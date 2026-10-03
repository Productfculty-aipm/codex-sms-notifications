# Copyable Codex instructions for SMS

Use these rules in the local instruction scope beside the SMS client. Replace every placeholder before enabling the route.

## Self-text routing

- When the user says “text me” or “text me a reminder,” route the request to the local Twilio API client.
- The default destination is the owner’s verified self number stored in the machine-local secret store as SMS_SELF_NUMBER.
- A clear immediate request authorizes one immediate self-text. Do not ask for the number again.
- A request with a future date and time uses the scheduling path. Never send a future reminder immediately as a fallback.
- Default timezone: America/Toronto. Resolve ambiguous timing before scheduling.
- Keep the body short and within one basic SMS segment where practical.

## Completion notifications

“Text me when this is done” authorizes one short self-text after the current task is actually complete and verified. Carry that requirement through continuations. Send it before the final chat reply. Do not text “done” for blocked, failed, paused, or unverified work. Repeated “text me” in one request means one notification.

## Client contract

Prepare a private packet outside the repository:

~~~json
{
  "id": "stable-task-request-id",
  "to": "${SMS_SELF_NUMBER}",
  "body": "The requested task is complete and verified.",
  "authorization": "self-text-request"
}
~~~

Call the local client with its authorized-send flag. The client must validate the destination, journal before POST, prevent duplicate requests, and read the provider receipt back. Keep a stable ID across retries and continuations.

## Provider truth

Report the provider state exactly:

- queued or accepted means Twilio accepted the request.
- scheduled means Twilio stored a future send.
- sent means Twilio submitted it to the carrier route.
- delivered means Twilio received a delivery confirmation.
- failed or undelivered means the provider or carrier reported a failure.

Never claim handset delivery from a queued or sent state. If a POST result is uncertain, inspect the saved request ID or provider SID before deciding what happened. Never retry an uncertain create request automatically.

## Security boundary

- Keep credentials in an owner-only local file, keychain, or secret manager.
- Keep private packets, journals, phone numbers, account SIDs, API key SIDs, and message SIDs out of Git.
- Use a dedicated restricted Twilio API key.
- Do not open the Twilio Console for normal sends; use it for one-time setup and diagnosis.
- Do not infer permission to contact another person from a self-text request.

This file describes routing behavior. It is not a credential store, background monitor, or guarantee of carrier delivery.

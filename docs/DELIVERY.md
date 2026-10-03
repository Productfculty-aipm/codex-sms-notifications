# Delivery, retries, and verification

SMS automation is complete only when the provider state has been reconciled. A local command finishing is not the same as a carrier delivery.

## State machine

~~~text
request -> validated -> journaled -> provider accepted -> provider readback
                                      |                         |
                                      |                         +--> queued / sent / delivered
                                      +--> definitive rejection +--> failed / undelivered
~~~

Record the originating request ID and, after a successful POST, the provider message SID. Keep those identifiers together in an owner-only journal.

## Immediate sends

1. Resolve the message and destination from the user request.
2. Confirm that the destination is the configured self number.
3. Validate the body, length, characters, and authorization.
4. Run sender or Messaging Service preflight.
5. Write the intent to the journal before POST.
6. POST through the Twilio REST API.
7. Read the message resource back by SID.
8. Report the exact returned status.

If the network fails after the POST may have reached Twilio, the result is unknown. Look up the request ID or message SID first. Do not create a second message just because the client did not receive a response.

## Scheduled reminders

Use a Messaging Service with a fixed SendAt value and an explicit timezone conversion. The local client should enforce Twilio’s current scheduling window and refuse a date it cannot hold safely.

A scheduled response proves that Twilio stored a future send. It does not prove handset delivery. Read the message status later and preserve the receipt.

Never silently fall back from scheduling to an immediate send. A missing Messaging Service or schedule field can change the meaning of the request.

## Retries

Retry only a definitive, understood rejection after correcting its cause. Examples include an invalid body or a configuration error that has been fixed.

Do not retry automatically after:

- a timeout after POST;
- a connection reset after POST;
- a provider response that was not read back;
- an unknown or missing message SID.

Inspect Twilio first. Stable request IDs make that inspection possible.

## Cancellation and replacement

Cancellation is a separate mutation. Reconcile the existing scheduled message, request cancellation, and read back canceled before creating a replacement. Do not run cancel and replacement in parallel.

A key revocation does not necessarily cancel already scheduled messages. Cancel future messages explicitly when that is part of the request, then verify each state.

## Reporting language

Use the provider’s vocabulary:

- **queued / accepted:** Twilio accepted the request;
- **scheduled:** Twilio stored a future send;
- **sent:** Twilio submitted the message to its carrier route;
- **delivered:** Twilio received a delivery confirmation;
- **failed / undelivered:** Twilio or the carrier reported a failure;
- **unknown:** the client cannot yet prove what happened.

Do not convert “queued” into “delivered,” and do not claim that a handset displayed a message without a delivery confirmation.

## Long-horizon work

Twilio scheduling is finite. For a date outside the supported provider window, use an explicitly configured follow-up or automation that revisits the request within the window. State that the reminder is not yet held by Twilio. Do not install a hidden daemon or guess a future completion time.

This route is a completion action in the working chat. It is not an always-running monitor unless a separate, visible automation has been configured and verified.

## Privacy and incident response

Keep credentials, real phone numbers, raw provider responses, request journals, and message bodies outside the public repository. If a secret appears in a commit, revoke it immediately, remove it from the working tree, and treat the old value as compromised.

Use a dedicated restricted API key for messaging. Keep voice, billing, and unrelated account permissions out of the key.

## Test plan

Before relying on the route:

- send one short self-test;
- read the returned message resource;
- exercise one definitive validation failure without posting;
- test duplicate request handling;
- schedule a near-term test and cancel it;
- verify that a second machine has its own credentials and journal.

Document the provider receipts privately. Do not paste them into a public issue, README, screenshot, or example.

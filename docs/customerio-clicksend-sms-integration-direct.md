# Customer.io + ClickSend SMS Integration (Direct, No Middleware)

## Goal

Send SMS campaigns from Customer.io through ClickSend using 4 phone numbers, each sending up to 10,000 SMS/day, for 40,000 SMS/day total — using direct API calls only, with no separate middleware service.

## How it works

Customer.io's Webhook action can call ClickSend's API directly. ClickSend's inbound webhook can call Customer.io's API directly. No service sits in between.

**Sending a message:**
Customer.io campaign → Webhook action → ClickSend Send SMS API → message delivered.

**Handling a reply (e.g. STOP):**
Recipient replies → ClickSend inbound webhook → Customer.io Track API → customer profile updated.

## Sending SMS from Customer.io

- Use Customer.io's Webhook action in the campaign or broadcast.
- Point it at ClickSend's Send SMS endpoint, using Basic Auth with your ClickSend username and API key.
- Build the message body using Liquid so it includes the customer's phone number, the message text, and which of the 4 numbers to send from.

## Number pool and rotation (static assignment)

Since there's no middleware to track live send counts, numbers are assigned to customers ahead of time instead of rotated in real time:

- Split your SMS audience into 4 even segments in Customer.io (for example, based on a modulo of customer ID).
- Assign one ClickSend number to each segment.
- Size each segment so it stays under 10,000 sends/day.

This keeps each number under its daily cap without needing a live counter. The trade-off is that number assignment is fixed per customer rather than dynamically load-balanced.

## Unsubscribe handling

- Turn on the same STOP/UNSUBSCRIBE keyword auto-reply on all 4 ClickSend numbers.
- Point all 4 numbers' inbound webhook at Customer.io's Track API, so a STOP on any number sets one shared attribute on that customer's profile (e.g. `sms_opted_out = true`).
- Before every send, only include customers where `sms_opted_out` is not true. This makes an opt-out on one number apply to all 4 automatically.

**Important requirement:** Customer.io's API identifies people by their Customer.io ID (not by phone number). For ClickSend's webhook to update the right profile directly, the customer's phone number needs to be usable as their Customer.io ID. If IDs are set to something else (like an internal user ID or email), a phone-number lookup step would be needed, which this direct setup does not include.

## Call forwarding

Call forwarding is a ClickSend account feature, not an API integration:

1. Confirm your ClickSend account has enough balance to cover the call-forwarding add-on fee.
2. Contact ClickSend support and request call forwarding for each of the 4 numbers, giving them your main business number as the destination.
3. ClickSend confirms and enables forwarding per number.
4. Test each number by calling it and confirming it rings through to the main line.

Note: call forwarding is only available for US and UK numbers, and for UK numbers it must be requested when the number is first purchased.

## What this approach gives up

- No live daily-cap enforcement — relies on fixed segment sizes staying under 10,000/day.
- No dynamic load-balancing across numbers.
- Opt-out sync only works cleanly if phone number is used as the Customer.io ID.

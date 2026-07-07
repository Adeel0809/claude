# Customer.io + ClickSend SMS Campaign Integration

## Goal

Send high-volume SMS campaigns through Customer.io using ClickSend as the SMS gateway, using **4 dedicated phone numbers**, each capped at **10,000 SMS/day**, for a combined capacity of **~40,000 SMS/day**.

## 1. Why a middleware layer is needed

Customer.io does not have a native, certified ClickSend integration (its built-in SMS support targets providers like Twilio/Vonage). The standard pattern for a non-native provider is:

```
Customer.io Campaign/Broadcast
        │  (Webhook action)
        ▼
Middleware service (small serverless function / internal API)
        │  (ClickSend Send SMS API)
        ▼
ClickSend  ──►  Carrier  ──►  Customer's phone
        │
        └── Inbound webhook (delivery receipts + STOP replies) ──► Middleware ──► Customer.io (update attributes)
```

The middleware is the piece that makes rotation, daily caps, and cross-number unsubscribes work — ClickSend and Customer.io don't coordinate on any of that by themselves.

**Components:**
- **Customer.io**: builds/segments the audience, triggers the send via a Webhook action in the campaign/broadcast, receives delivery/opt-out status back as customer attributes and events for reporting.
- **Middleware** (e.g., AWS Lambda, Cloudflare Worker, or a small internal service): owns rotation logic, daily send counters, and the master suppression list.
- **ClickSend**: owns the 4 dedicated numbers, sends the SMS, and posts delivery receipts + inbound replies (STOP, etc.) to a webhook.

## 2. Number pool & volume plan

| Number | Daily cap | Purpose |
|---|---|---|
| Number 1 | 10,000 | Rotation pool |
| Number 2 | 10,000 | Rotation pool |
| Number 3 | 10,000 | Rotation pool |
| Number 4 | 10,000 | Rotation pool |
| **Total** | **40,000/day** | |

**Open item to confirm with ClickSend before committing to this plan:** sustained throughput of 10,000/day on a single number depends on the number type and (for the US) 10DLC brand/campaign trust score. Standard 10DLC long codes at lower trust tiers may be throttled by carriers below this. Before launch, confirm with ClickSend/your carrier registration that each number's registered 10DLC campaign supports 10,000 msgs/day, or use toll-free numbers (higher default throughput, faster to provision) instead of long codes.

## 3. Number rotation

Rotation is handled in the middleware, not in ClickSend or Customer.io (neither has a "rotate sender number" feature natively).

**Approach: deterministic (sticky) hashing, not pure round-robin.**
- Compute `bucket = hash(contact_id) % 4` and always send that contact's messages from the same number.
- **Why sticky, not round-robin:** if a customer's replies (including STOP) and threading need to stay on one number, and it keeps a given customer's message history consistent from their point of view. Round-robin would cause the same customer to receive texts from a different number every time, which looks spammy and breaks reply-based flows (e.g., "reply YES").

**Daily cap enforcement:**
- Middleware keeps a per-number send counter (e.g., in Redis/DynamoDB), reset at local midnight.
- Before sending, check the assigned number's counter:
  - Under cap → send normally.
  - At cap → fail over to the next-least-loaded number under cap (log this as an overflow event so volume distribution can be reviewed).
- If a number is flagged/paused (e.g., carrier filtering, high complaint rate), remove it from the rotation pool until resolved and rebalance the hash across the remaining numbers.

## 4. Unsubscribe / opt-out handling (must be centralized, not per-number)

This is the part that needs the most care. **ClickSend's built-in opt-out handling is scoped to the list/number a STOP was sent to — it does not automatically apply across your other 3 numbers or other Customer.io segments.** If you rely only on ClickSend's default behavior, a contact could STOP on Number 1 and still receive messages from Numbers 2–4.

**Design: single source of truth for opt-out, checked before every send, regardless of which number is used.**

1. Configure the same STOP/UNSUBSCRIBE/END keyword auto-reply on **all 4 numbers** in ClickSend, and point all 4 numbers' inbound webhook at the same middleware endpoint.
2. When the middleware receives an inbound STOP (from any of the 4 numbers):
   - Look up the contact by phone number.
   - Set a single suppression flag on the Customer.io profile, e.g. `sms_opted_out = true` (via the Customer.io API), and remove them from any active SMS segments/campaigns.
   - Optionally also push the number into ClickSend's account-level blocklist so even out-of-band sends (e.g., someone manually sending via the ClickSend dashboard) are blocked.
3. Before every send (regardless of which of the 4 numbers is selected by rotation), the middleware checks `sms_opted_out` on the Customer.io profile first. If true, the send is skipped and logged — this is what makes the opt-out effectively global across all 4 numbers.
4. Re-subscription: if the business supports opt-in again (e.g., contact texts START), mirror the same flow to clear `sms_opted_out`.

This keeps Customer.io as the system of record for consent, which also makes it easy to build a "do not SMS" segment for anyone else building future campaigns.

## 5. Call forwarding (voice calls to the 4 numbers → main line)

SMS numbers used for campaigns will also receive inbound voice calls from people calling back. These need to forward to your main business number.

**ClickSend specifics:**
- Call forwarding is **not enabled by default** and is currently only available for **US and UK** numbers.
- It's an **add-on service that ClickSend support enables manually per number** — it isn't self-serve in the dashboard.
- For UK numbers specifically, forwarding can only be set up **at the time the number is provisioned** — it cannot be added to a number you already own.

**Setup steps (per number):**
1. Ensure the ClickSend account has sufficient prepaid balance to cover the monthly call-forwarding add-on fee for each number.
2. Contact ClickSend support and request call forwarding for each of the 4 numbers, providing the destination number (your main business line) for all four.
3. ClickSend confirms whether each number supports forwarding and enables it.
4. Test each of the 4 numbers by calling them and confirming the call rings through to the main line.

**Practical note:** if any of the 4 numbers are being newly purchased (rather than already owned), request call forwarding to be configured at time of purchase — this avoids the UK "can't add forwarding after the fact" limitation and avoids a second round-trip with support.

## 6. Compliance notes

- Ensure the audience being messaged has valid SMS opt-in consent (TCPA in the US, equivalent regs elsewhere) before the campaign sends — this is independent of the opt-out mechanism above and should be enforced by segment criteria in Customer.io.
- Include a clear sender identity and opt-out instruction ("Reply STOP to unsubscribe") in the first message of any campaign.
- Respect quiet hours for the recipient's timezone if messaging across regions.
- If sending in the US at this volume, register the brand/numbers under 10DLC (see the volume note in §2) to avoid carrier filtering.

## 7. Monitoring

- Track daily send count per number vs. the 10,000 cap (dashboard or alert when a number crosses ~90%).
- Track delivery/failure rate per number — a sudden drop usually means carrier filtering on that number.
- Track opt-out rate per campaign/number to catch content or frequency problems early.
- Surface all of the above back into Customer.io as events/attributes so campaign performance can be segmented by number.

## 8. Open items before implementation

- [ ] Confirm with ClickSend that 10,000/day/number is realistic for the chosen number type/country (10DLC trust tier vs. toll-free).
- [ ] Decide where the middleware lives (serverless function vs. existing backend service) and who owns/maintains it.
- [ ] Confirm the main business number that all 4 numbers should forward voice calls to.
- [ ] Decide on the STOP/re-subscribe (START) keyword wording, consistent across all 4 numbers.

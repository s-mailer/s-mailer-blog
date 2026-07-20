---
layout: post
title: "Inbound Webhooks & Triggers: How to receive and route SMS messages with S-Mailer"
date: 2026-07-14 07:00:00 +0200
categories: guides
author: S-Mailer Team
tags: [webhooks, inbound, sms, triggers, integration]
---

Sending messages is only half of a messaging platform. The other half is what
happens when someone replies — a customer answering `YES` to a confirmation, an
M-Pesa payment notification landing on your number, a support request arriving at
2 a.m. **Inbound webhooks** are how S-Mailer hands those messages to your
application in real time.

This guide covers the payload, the routing rules, and how to wire it all up.

## What is an inbound webhook?

When a message arrives on one of your senders, S-Mailer stores it and then makes
an HTTP `POST` to a URL you control, with the message as JSON. You don't poll, you
don't run a cron job — your endpoint gets called within a moment of the message
landing on the device.

That single mechanism unlocks a lot: confirm payments the instant the mobile money
SMS arrives, open a support ticket from a text message, let a customer reply
`CANCEL` to an order and have your backend act on it.

**The payload is the same for every channel.** SMS and WhatsApp today, email
tomorrow — `from` and `to` carry whatever addresses that channel uses, and a
`channel` field tells you how to read them. One parser handles all of them, so
adding a channel later doesn't mean rewriting your receiver.

---

## The webhook payload

Every delivery looks like this:

```json
{
  "body":             "PAY 1500 order 4471",
  "channel":          "sms",
  "client_id":        "1a2b3c4d-5e6f-4071-8293-a4b5c6d7e8f9",
  "from":             "+258840000000",
  "id":               "9f2c1a7e-4b3d-4c8f-9e21-5a7b8c0d1e2f",
  "payment_required": false,
  "received_at":      "2026-07-14T15:30:00+00:00",
  "sender_id":        "3c6f2b10-8d44-4a19-b7e5-2f9a1c4d6e88",
  "to":               "+258849999999"
}
```

Keys arrive **alphabetically ordered**, as shown. Don't depend on it — parse by
name, not by position — but that is what lands on the wire.

| Field | What it is | What you do with it |
|---|---|---|
| `id` | UUID of the stored inbound message | Your idempotency key — store it, and ignore a message you've already seen. |
| `from` | The address that sent the message | The customer's number. Look up their account, reply to them. |
| `to` | The address that received it (your sender) | Which of your numbers it landed on — useful when you run several. |
| `channel` | `sms`, `whatsapp`, `email`, … | Tells you how to interpret `from`/`to` and how to reply. |
| `body` | The message content | The part you actually parse: keywords, amounts, reference codes. |
| `received_at` | RFC 3339 UTC timestamp | Ordering and auditing. Don't trust arrival order of HTTP requests — sort on this. |
| `sender_id` | UUID of the sender it arrived on | Route by sender when one backend serves several numbers. |
| `client_id` | UUID of the owning client | Your account id. Handy when one endpoint serves multiple S-Mailer accounts. |
| `payment_required` | `true` when your balance was too low to charge for the message | The delivery is **locked**: `from` and `body` come through as `null`. See [Locked messages](#locked-messages-when-your-balance-runs-out). |

Deliberately absent: device ids, SIM slots, and other transport details. They mean
nothing outside one provider, so they stay on the stored message's metadata rather
than in your contract.

### Locked messages: when your balance runs out

An inbound message is never dropped over billing — but it is not handed over for
free either. If a message arrives when your balance is too low to be charged for
it, S-Mailer stores it in full and sends your webhook a **locked notice** instead
of the message:

```json
{
  "body":             null,
  "channel":          "sms",
  "client_id":        "017cb3ea-af74-5886-f818-b8a30743e222",
  "from":             null,
  "id":               "9f2c1a7e-4b3d-4c8f-9e21-5a7b8c0d1e2f",
  "payment_required": true,
  "received_at":      "2026-07-14T15:30:00+00:00",
  "sender_id":        "3f1b2c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
  "to":               "+258840000001"
}
```

**The shape never changes.** Every field is always present — a locked delivery
only nulls the ones you have not paid for, so you parse one payload and branch on
`payment_required` alone, never on which fields happen to exist.

`from` is withheld along with `body`: knowing who is contacting you, and how
often, is worth something on its own. `to`, `channel` and `received_at` describe
your own sender rather than the person who wrote to you, so they stay. The
content is safe in S-Mailer and readable in the dashboard under
**Messages → Inbound**, where the message is marked `Locked`.

To get the real delivery: top up, then hit **Re-trigger** on that message. S-Mailer
charges you for it and POSTs the same payload with `from` and `body` filled in and
`payment_required: false`. Re-triggering twice is refused, so you are never charged
twice for one message.

So a receiver needs one branch:

```js
if (msg.payment_required) {
  // Locked: the body is not here. Top up and re-trigger from the dashboard.
  alertOps(`inbound ${msg.id} is locked — S-Mailer balance is empty`);
  return;
}
```

---

## Use cases

**Payment confirmation (M-Pesa / eMola).** The mobile money confirmation SMS lands
on your gateway number, your webhook parses the amount and reference out of `body`,
and the order is marked paid — no human refreshing a dashboard.

**Customer support and ticketing.** Inbound message in, ticket out. `from` is the
requester, `body` is the ticket text, `received_at` starts your SLA clock.

**Order notifications.** A customer texts `STATUS 4471` and your backend replies
with the order state, using the S-Mailer send API to answer on the same number.

**Two-factor and confirmations.** You send "Reply YES to confirm"; the reply comes
back through the webhook and your flow advances.

**Multi-tenant routing.** One phone number, several applications behind it. That's
what triggers are for.

---

## Triggers: keyword routing

A webhook on your account catches *everything* that arrives on *any* of your
senders. Triggers are finer-grained: **a keyword plus a callback URL, attached to a
specific sender.**

The rule is simple: **if the keyword appears anywhere in the message body, the
trigger fires.** It's a case-insensitive substring match — no prefix requirement, no
regex, no exact-match rules to fight with. Keyword `pay` matches `PAY 1500`,
`Payment done`, and `i want to pay`.

That looseness is a feature for catching real-world messages, and a trap if you pick
a keyword like `a`. Choose keywords distinctive enough that they can't show up by
accident.

### Shared senders

Triggers exist because **one phone number can be shared between clients.** Each
client puts their own keywords on that shared sender, and each gets only the
messages meant for them:

```
                    +258 84 999 9999  (shared sender)
                              |
              inbound message arrives
                              |
                   +----------+----------+
                   |                     |
          body contains "PAY"    body contains "SUPPORT"
                   |                     |
                   v                     v
        Client A's callback      Client B's callback
        https://a.example/pay    https://b.example/tickets
```

A message reading `PAY 1500 order 4471` goes to Client A. `SUPPORT my line is
down` goes to Client B. A message matching **both** keywords goes to **both** —
matching isn't exclusive, and the first match doesn't stop the rest.

### How the two combine

For each inbound message, S-Mailer builds the list of destinations:

1. Your account's webhook URL, if you've set one.
2. Every **active** trigger on the receiving sender whose keyword appears in the body.

The list is **deduplicated by URL**. Point your account webhook and a trigger
callback at the same URL and you get one POST, not two. Point them at different
URLs and each gets its own copy — so if both handlers write to the same database,
dedupe on `id`.

---

## Setting it up in Dash

Everything lives under **Dashboard → Webhooks**.

That page also hosts a second field — the **Status Webhook** — for delivery-status
updates on messages you *send*, described below.

**Set your webhook URL.** Paste an `http(s)` URL into the webhook field and save.
Every message arriving on any of your senders is POSTed there. Clear the field to
turn it off.

**Add a trigger.** Pick a sender (the dropdown lists the senders you own *and* the
ones shared with you), type a keyword, give a callback URL, and save. You can pause
a trigger without deleting it, and re-activate it later — a paused trigger never
fires. One keyword per sender per client: adding the same keyword twice on the same
sender is rejected.

**Test it.** Send an SMS containing your keyword to the sender number from any
phone. During development, point the callback at a tunnel
([ngrok](https://ngrok.com), [webhook.site](https://webhook.site)) so you can see the
raw request. If nothing arrives, check that the trigger is active, that the keyword
really appears in the body you sent, and that your endpoint is reachable from the
public internet.

---

## Status webhooks: delivery updates on messages you send

Inbound webhooks fire on messages that arrive. The **status webhook** is the
mirror image: it fires when a message you *sent* changes state — an Android gateway
relaying a carrier delivery report, or WhatsApp acknowledging that a message reached
the handset and was opened.

Set the default **Status Webhook URL** in the same place — **Dashboard →
Webhooks**, in its own field next to the inbound webhook. You can also override it
for a single message by passing `webhook_url` on the
[send request](https://api.mailer.smartek.co.mz). Set neither and no status webhook
fires.

The payload is its own shape — a status change, not a message:

```json
{
  "error_code": null,
  "message_id": "7d3e9b21-6c44-4f18-a2e5-1b9c8d0f4a77",
  "recipient":  "+258840000000",
  "status":     { "current": "delivered", "old": "sent" },
  "timestamp":  "2026-07-14T15:31:04+00:00"
}
```

| Field | What it is |
|---|---|
| `message_id` | UUID of the message you sent — your correlation key back to the send call. |
| `recipient` | The address the update is about. |
| `status.old` | Status before this change (e.g. `sent`). |
| `status.current` | Status after — see the table below. |
| `error_code` | Why it failed, as reported by the provider. `null` on a delivery. |
| `timestamp` | When the change was recorded, RFC 3339 UTC. |

### The statuses a message walks

| Status | Meaning | Channels |
|---|---|---|
| `sent` | Handed to the network. | all |
| `delivered` | Reached the recipient's device. | all |
| `read` | The recipient opened it. | WhatsApp only |
| `failed` | The provider gave up. `error_code` says why. | all |

`read` never arrives on SMS — the carrier has no way to know — and it only arrives
on WhatsApp when the recipient has read receipts switched on. A message that stops
at `delivered` is normal, not a fault.

**Updates only ever move forward.** Providers do not order their callbacks: WhatsApp
can hand over the *delivered* ack after the *read* one. S-Mailer drops any update
that would walk a message backwards, so you never get a webhook telling you a read
message un-read itself. `failed` is terminal — nothing supersedes it.

**A status can fire more than once per message** — one webhook per recipient per
change. Dedupe on `message_id` *plus* `recipient` *plus* `status.current`, not on
`message_id` alone.

Same handling rules as inbound: answer `2xx` fast and protect the URL (it isn't
signed). See [turning an Android phone into a
gateway]({% post_url 2026-07-05-turn-your-android-into-a-gateway %}) for where the
SMS delivery reports come from, and [connecting
WhatsApp]({% post_url 2026-07-20-connect-whatsapp-to-s-mailer %}) for the WhatsApp
side.

---

## Handling a webhook

The contract is short: **read the JSON, answer `2xx`, do the work afterwards.**

### curl (to see the shape)

```bash
curl -X POST https://your-app.example/webhooks/s-mailer \
  -H "Content-Type: application/json" \
  -d '{
        "body": "PAY 1500 order 4471",
        "channel": "sms",
        "client_id": "1a2b3c4d-5e6f-4071-8293-a4b5c6d7e8f9",
        "from": "+258840000000",
        "id": "9f2c1a7e-4b3d-4c8f-9e21-5a7b8c0d1e2f",
        "payment_required": false,
        "received_at": "2026-07-14T15:30:00+00:00",
        "sender_id": "3c6f2b10-8d44-4a19-b7e5-2f9a1c4d6e88",
        "to": "+258849999999"
      }'
```

### PHP

```php
<?php
// POST /webhooks/s-mailer
$msg = json_decode(file_get_contents('php://input'), true);

if (!$msg || empty($msg['id'])) {
    http_response_code(400);
    exit;
}

// Acknowledge first — the work happens after the response.
http_response_code(204);
fastcgi_finish_request();

if (alreadyProcessed($msg['id'])) {
    return; // duplicate delivery, nothing to do
}
markProcessed($msg['id']);

if (!empty($msg['payment_required'])) {
    // Locked — there is no body to read. Top up, then re-trigger it.
    error_log("inbound {$msg['id']} is locked — S-Mailer balance is empty");
    return;
}

if (stripos($msg['body'], 'PAY') !== false) {
    confirmPayment($msg['from'], $msg['body']);
}
```

### Node.js (Express)

```js
app.post('/webhooks/s-mailer', express.json(), async (req, res) => {
  const msg = req.body;
  if (!msg?.id) return res.sendStatus(400);

  res.sendStatus(204);              // ack immediately

  if (await seen(msg.id)) return;   // idempotency
  await remember(msg.id);

  if (msg.payment_required) {
    // Locked — there is no body to read. Top up, then re-trigger it.
    console.warn(`inbound ${msg.id} is locked — S-Mailer balance is empty`);
    return;
  }

  if (msg.body.toLowerCase().includes('pay')) {
    await confirmPayment(msg.from, msg.body);
  }
});
```

### Python (Flask)

```python
@app.post("/webhooks/s-mailer")
def s_mailer_inbound():
    msg = request.get_json(silent=True) or {}
    if not msg.get("id"):
        return "", 400

    if seen(msg["id"]):               # idempotency
        return "", 204
    remember(msg["id"])

    if msg.get("payment_required"):
        # Locked — there is no body to read. Top up, then re-trigger it.
        app.logger.warning("inbound %s is locked — S-Mailer balance is empty", msg["id"])
        return "", 204

    # Hand off to a worker; keep the response fast.
    queue.enqueue(process_inbound, msg)
    return "", 204
```

---

## Security & best practices

**Answer fast, work later.** S-Mailer waits **10 seconds** for a response and then
gives up. Delivery is **fire-and-forget: there is no retry.** A slow endpoint or a
`500` means that message is gone from your side — it stays stored in S-Mailer, but
your handler will not be called again. So acknowledge with a `2xx` first and push
the real work onto a queue or a background task.

**Deduplicate on `id`.** Two destinations can point at the same handler, and a
proxy in front of you may replay a request. Keep a table of processed message ids
and make the second delivery a no-op.

**Protect the endpoint.** S-Mailer does not sign webhook requests today — the POST
carries only `Content-Type: application/json`, no signature header. Until it does,
your callback URL *is* the secret, so treat it like one:

- Serve it over **HTTPS**, never plain HTTP.
- Use an **unguessable path or query token**
  (`https://your-app.example/hooks/s-mailer/8f3a…`) and reject anything that doesn't
  present it.
- Validate the payload before acting on it — check that `client_id` and `sender_id`
  are yours, and never trust `body` straight into a shell, a query, or an eval.
- If your infrastructure allows it, restrict the endpoint to S-Mailer's egress IP.

**Handle `payment_required`.** When it's `true` the delivery is locked, because your
balance was too low to charge for the message (see
[Locked messages](#locked-messages-when-your-balance-runs-out)). The payload keeps
its usual shape, so check the flag *before* you read anything — `from` and `body`
arrive as `null`, and code that reaches straight for `body` will fall over on a
null rather than a missing key. Treat it as an operational alert: log it, page
whoever owns the account, and top up. Then re-trigger the message from the
dashboard to receive it in full. Better still, set a low-balance alert under
**Dashboard → Profile** and never see a locked delivery at all.

**Keywords are substrings.** `pay` inside `repayment` will fire your trigger. Pick
keywords with enough shape to avoid accidents, and validate the full body in your
handler rather than trusting that the match was meaningful.

---

## Next steps

You now have the whole picture: one channel-agnostic payload, an account-wide
webhook for everything, and keyword triggers for routing off shared senders.

To get started:

1. Register a sender — [turn an Android phone into a
   gateway]({% post_url 2026-07-05-turn-your-android-into-a-gateway %}) if you don't
   have one yet.
2. Stand up an endpoint that answers `2xx` and queues its work.
3. Set the webhook URL under **Dashboard → Webhooks**, text your number, and watch
   it arrive.
4. Add triggers once you need to split traffic across applications or tenants.

Then close the loop: reply to inbound messages with the
[send API](https://api.mailer.smartek.co.mz) and you have a full two-way
conversation running on a phone number.

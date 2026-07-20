---
layout: post
title: "Now available: connect WhatsApp to S-Mailer"
date: 2026-07-20 04:00:00 +0200
categories: announcements
author: S-Mailer Team
---

Big news: you can now **send and receive WhatsApp messages straight from
S-Mailer**, using your own WhatsApp number. No special hardware, no lengthy
approvals — you connect in under two minutes by scanning a QR code, exactly like
adding WhatsApp Web to your browser. Here's everything you need to get started.

## What you'll need

- An S-Mailer account with a token balance. [Sign up here](https://mailer.smartek.co.mz).
- A phone with the **WhatsApp number** you want to send and receive from.

---

## Step 1 — Register a WhatsApp sender

1. Log in to the [S-Mailer dashboard](https://mailer.smartek.co.mz).
2. Go to **Senders → Register New Sender**.
3. In **Provider**, choose **WhatsApp (QR-paired)**.
4. Give it a name you'll recognise (for example, *Support line*).
5. Leave **Config JSON** as the default `{}` — nothing else is needed here.
6. Click **Register Sender**.

Your sender is created right away and appears in your list, ready to connect.

## Step 2 — Scan the QR code to connect

1. Click your new sender to open it. You'll see **Scan to connect** and a QR
   code (it refreshes on its own).
2. On your phone, open **WhatsApp → Settings → Linked devices → Link a device**.
3. Point your phone at the QR code on screen.

That's it — the panel flips to **Connected** by itself, and your real WhatsApp
number appears. Your sender's status also shows **Active** back on the Senders
page.

## Step 3 — Send your first WhatsApp message

Create an API key with the **Send** permission for the WhatsApp channel
(**API Keys → New key**), then call the API:

```bash
curl -X POST https://api.mailer.smartek.co.mz/api/v1/send \
  -H "X-Client-ID: <your-client-id>" \
  -H "X-Client-Secret: <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
        "channel": "whatsapp",
        "recipient": "+258840000000",
        "content": "Hello from S-Mailer on WhatsApp!"
      }'
```

The message goes out from your connected number. You can send plain text
straight away — right from the API or the dashboard.

---

## Receiving messages (optional)

Want replies to flow back to your systems? On the sender's page, turn on
**Inbound messages**. While it's on, every WhatsApp message your number receives
is forwarded to your webhook, and tokens are deducted per received message at
your plan's current inbound rate. While it's off, incoming messages are ignored
and nothing is charged — so it's entirely opt-in.

For the webhook payload and how to route replies with keyword triggers, see the
[inbound & status webhooks guide]({% post_url 2026-07-14-inbound-webhooks-triggers %}).

## Good to know

- **Keep your phone online.** Like any WhatsApp linked device, your sender stays
  connected as long as your phone is reachable. If it disconnects, just reopen
  the sender and scan again.
- **It's your own number.** Send responsibly and within normal WhatsApp usage —
  the reputation of the number is yours.
- **Disconnect anytime.** Deleting the sender unlinks the device cleanly.

WhatsApp joins SMS as a fully supported channel in S-Mailer — same dashboard,
same API, one more way to reach your customers. Happy messaging!

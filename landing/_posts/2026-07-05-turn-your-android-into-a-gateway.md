---
layout: post
title: "How to turn your Android into a powerful gateway for S-Mailer"
date: 2026-07-05 09:00:00 +0200
categories: guides
author: S-Mailer Team
---

One of the most powerful features of S-Mailer is that you don't need any special
hardware to start sending SMS. **Any Android phone can become a sending gateway.**
This guide walks you through it, step by step.

## What you'll need

- An Android phone (Android 8.0 / API 26 or newer) with an active SIM.
- An S-Mailer account with a token balance. [Sign up here](https://mailer.smartek.co.mz).
- The **S-Mailer** Android app — [**download the APK**](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer.apk).

---

## Step 1 — Install the app

**[⬇️ Download the S-Mailer app (APK)](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer.apk)**

Install it on the phone you want to use as a gateway. Because it is installed
outside the Play Store, allow installs from your browser or file manager when
prompted. Open the app and grant the SMS and phone permissions it asks for — it
needs these to read your SIMs and send messages.

![Install and grant permissions](/assets/img/guide-01-permissions.jpg)
*The permissions screen on first launch.*

## Step 2 — Copy the pairing payload

On first launch the app generates a **pairing payload** — a small block of JSON
that identifies this device (its ID, a one-time pairing code, and its SIMs). Tap
**Copy** to put it on your clipboard.

![Copy the pairing payload](/assets/img/guide-02-pairing.jpg)
*The pairing screen with the Copy button.*

## Step 3 — Register the sender in the dashboard

1. Log in to the [S-Mailer dashboard](https://mailer.smartek.co.mz).
2. Go to **Senders → Register sender**.
3. Choose the **Android SMS Gateway** option.
4. Paste the pairing payload you copied from the app.
5. Save.

S-Mailer then pairs your device to the new sender.

![Register the sender](/assets/img/guide-03-register.jpg)
*The sender registration form in the dashboard.*

## Step 4 — Confirm it is paired

Back on the phone, the app switches to its dashboard and shows **"Paired with
&lt;your company&gt;"**. Keep the app running and the phone online — it now holds a
live connection to S-Mailer and waits for messages to send.

![Paired and ready](/assets/img/guide-04-paired.jpg)
*The app showing the paired status.*

## Step 5 — Send your first message

Create an API key (**API Keys → New key**, with the **Send** permission for the
SMS channel), then call the API:

```bash
curl -X POST https://api.mailer.smartek.co.mz/api/v1/send \
  -H "X-Client-ID: <your-client-id>" \
  -H "X-Client-Secret: <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
        "channel": "sms",
        "recipient": "+258840000000",
        "content": "Hello from my Android gateway!"
      }'
```

Your phone receives the send request over its live connection and sends the SMS
through its SIM. The delivery status flows back to your dashboard — and, as of the
latest app, all the way back to `delivered` (see below).

---

## Choose which SIM sends (app 0.1.3)

Got two SIMs in the gateway phone? By default S-Mailer sends through the phone's
preferred SIM. As of **app 0.1.3** you can pick the sending SIM yourself:

1. Open the app and go to **Settings**.
2. Under **Send SIM**, choose **System default** or a specific slot (each active
   SIM is listed with its carrier and number).
3. Tap **Save**.

Every message this gateway sends from then on goes out on the SIM you picked —
handy when one line has a better SMS bundle or a specific sender identity. Grab
the [latest APK](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer.apk)
to get the picker.

## From `sent` to `delivered`: delivery reports

Android hands the app a **delivery report** once the carrier confirms the SMS
reached (or failed to reach) the recipient's handset. The app relays that report
back through the connection, so a message can move past `sent`:

```
phone radio → app (delivery report) → co1 gateway → S-Mailer Core → your status webhook
```

Core flips the recipient from `sent` to `delivered` (or `failed`) and, if you've
configured a **status webhook**, POSTs you the change. Set the status webhook URL
under **Dashboard → Webhooks** — it's separate from the inbound webhook, and you
can also override it per message with a `webhook_url` on the send request. See the
[inbound & status webhooks guide]({% post_url 2026-07-14-inbound-webhooks-triggers %})
for the payload.

---

## Tips for a reliable gateway

- **Keep the phone charged and online.** A device on Wi-Fi or mobile data with a
  stable connection delivers fastest.
- **Watch your SIM limits.** Carriers may rate-limit or block high SMS volumes —
  spread load across multiple devices for scale.
- **`sent` means handed to the radio** — but Android delivery reports can update
  it to `delivered`. The gateway relays the carrier's delivery report back through
  co1 to S-Mailer Core, which flips the status and fires your status webhook (see
  [From `sent` to `delivered`](#from-sent-to-delivered-delivery-reports)). Delivery
  reports depend on the carrier honouring them, so treat `delivered` as best-effort.

That's it — you've turned an ordinary Android phone into a programmable SMS
gateway. Happy sending!

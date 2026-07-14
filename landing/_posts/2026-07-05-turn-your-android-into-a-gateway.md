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
through its SIM. The delivery status flows back to your dashboard.

---

## Tips for a reliable gateway

- **Keep the phone charged and online.** A device on Wi-Fi or mobile data with a
  stable connection delivers fastest.
- **Watch your SIM limits.** Carriers may rate-limit or block high SMS volumes —
  spread load across multiple devices for scale.
- **`sent` is not `delivered`.** On the Android gateway, `sent` means the message
  was handed to the radio. For guaranteed delivery reports, use a carrier-grade
  sending option.

That's it — you've turned an ordinary Android phone into a programmable SMS
gateway. Happy sending!

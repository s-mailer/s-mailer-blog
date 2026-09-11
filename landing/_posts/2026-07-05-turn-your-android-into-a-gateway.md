---
layout: post
title: "How to turn your Android into a powerful gateway for S-Mailer"
date: 2026-07-05 09:00:00 +0200
last_modified_at: 2026-09-11 09:00:00 +0200
categories: guides
author: S-Mailer Team
---

One of the most powerful features of S-Mailer is that you don't need any special
hardware to start sending SMS. **Any Android phone can become a sending gateway.**
This guide walks you through it, step by step.

## What you'll need

- An Android phone (Android 8.0 / API 26 or newer) with an active SIM.
- An S-Mailer account with a token balance. [Sign up here](https://mailer.smartek.co.mz).
- The **S-Mailer Gateway** app — there are two, one per provider (`sms-co1` and
  `sms-co2`), and which one you install depends on the sender you created in the
  dashboard. [Pick yours below](#step-1--install-the-right-app).

---

## Step 1 — Install the right app

There are **two** gateway apps, and they are not interchangeable. When you create
a sender in the dashboard you pick how the phone will be reached, and the sender's
page then shows that choice as a **provider**: `sms-co1` or `sms-co2`. Each app
speaks one of the two, and pairing simply never completes in the wrong one.

**Open your sender in the dashboard, read the provider, and download the file with
the same name.**

| Your sender says | Download | On your phone |
|---|---|---|
| `sms-co1` | [⬇️ s-mailer-sms-co1.apk](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer-sms-co1.apk) | 🔵 **S-Mailer Gateway**, blue icon |
| `sms-co2` | [⬇️ s-mailer-sms-co2.apk](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer-sms-co2.apk) | 🟠 **S-Mailer Gateway 2**, orange icon |

`sms-co2` is the one to prefer for a new sender: it keeps working when the phone
drops off the network for a while and reconnects, where `sms-co1` needs the app to
hold a live connection. Either way, install the one your sender was created with —
you cannot swap an existing sender to the other app.

Once installed, the two look identical when opened, so **the icon colour is how you
tell them apart** on your home screen. You can install both on the same phone, one
sender each, and they will not interfere with each other.

Install on the phone you want to use as a gateway. Because it is installed outside
the Play Store, allow installs from your browser or file manager when prompted.
Open the app and grant the SMS and phone permissions it asks for — it needs these
to read your SIMs and send messages.

> **Already using the old single app?** This release replaces it with the two
> above, and Android treats them as new apps rather than an update — so install
> the one you need, pair it, and only then uninstall the old one. If Android
> refuses an install with *"app not installed"* or *"package conflicts with an
> existing package"*, uninstall the old app first.
>
> Once you are on one of the new apps, later updates to it install straight over
> the top and keep your pairing and message history.

![Install and grant permissions](/assets/img/guide-01-permissions.jpg)
*The permissions screen on first launch.*

## Step 2 — Copy the pairing payload

On first launch the app generates a **pairing payload** — a small block of JSON
that identifies this device (its ID, a one-time pairing code, and its SIMs). Tap
**Copy** to put it on your clipboard.

![Copy the pairing payload](/assets/img/guide-02-pairing.jpg)
*The pairing screen with the Copy button.*

## Which gateway should you pick? sms-co1 vs sms-co2 {#which-gateway}

S-Mailer gives you **two Android gateway options**, and you choose which one when
you register the sender. The app and the pairing payload are **exactly the same**
for both — the only difference is how the phone waits for work. One device can
even back one of each at the same time.

### sms-co2 — set it and forget it

- ✅ **The app does not need to stay open.** The phone can be locked, the app
  swiped away or asleep — S-Mailer wakes it when there's a message to send.
- ✅ **Survives Android's battery management**, so it's the better choice for an
  unattended phone that just sits there being a gateway.
- ⚠️ **Slightly higher latency**: if the phone has been idle, the first message
  can take a few extra seconds while it wakes up.
- ⚠️ Needs a **reliable internet connection** to be reachable.

### sms-co1 — instant, but keep it running

- ✅ **Lowest latency** — the phone holds a live connection, so it sends the very
  moment a request arrives.
- ⚠️ **The app must stay open and connected in the foreground**, and the phone
  awake and online. Close the app or let the phone sleep and sending pauses until
  you reopen it.
- ⚠️ Needs the **battery-optimisation exemption** (the app prompts for it) and, in
  practice, more babysitting for a phone meant to run unattended.

**Rule of thumb:** for a phone that will run on its own, pick **sms-co2**. Pick
**sms-co1** only when the phone is attended and you want the snappiest possible
sends.

## Step 3 — Register the sender in the dashboard

1. Log in to the [S-Mailer dashboard](https://mailer.smartek.co.mz).
2. Go to **Senders → Register sender**.
3. Choose a gateway — **sms-co1** or **sms-co2** (see
   [Which gateway?](#which-gateway) above).
4. Paste the pairing payload you copied from the app.
5. Save.

S-Mailer then pairs your device to the new sender.

![Register the sender](/assets/img/guide-03-register.jpg)
*The sender registration form in the dashboard.*

## Step 4 — Confirm it is paired

Back on the phone, the app switches to its dashboard and shows **"Paired with
&lt;your company&gt;"**. Keep the phone online and waiting for messages to send. If
you paired on **sms-co1**, leave the app open in the foreground; on **sms-co2** you
can close it — S-Mailer will wake it when there's work.

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

Your phone receives the send request and sends the SMS through its SIM. The
delivery status flows back to your dashboard — and, as of the latest app, all the
way back to `delivered` (see below).

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
the latest APK ([sms-co1](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer-sms-co1.apk) ·
[sms-co2](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer-sms-co2.apk))
to get the picker.

## From `sent` to `delivered`: delivery reports

Android hands the app a **delivery report** once the carrier confirms the SMS
reached (or failed to reach) the recipient's handset. The app relays that report
back through the connection, so a message can move past `sent`:

```
phone radio → app (delivery report) → gateway → S-Mailer Core → your status webhook
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
  it to `delivered`. The gateway relays the carrier's delivery report back to
  S-Mailer Core, which flips the status and fires your status webhook (see
  [From `sent` to `delivered`](#from-sent-to-delivered-delivery-reports)). Delivery
  reports depend on the carrier honouring them, so treat `delivered` as best-effort.

That's it — you've turned an ordinary Android phone into a programmable SMS
gateway. Happy sending!

---
layout: post
title: "Control exactly what your Android gateway forwards"
date: 2026-07-26 09:00:00 +0200
categories: guides
author: S-Mailer Team
---

When you turn an Android phone into an [S-Mailer gateway]({% post_url 2026-07-05-turn-your-android-into-a-gateway %}),
it can forward the SMS it receives back to your S-Mailer account — perfect for
capturing replies and running two-way flows. But a gateway SIM also receives a lot
of noise: bank OTPs, carrier notices, and marketing blasts you don't want in your
account. **As of app 0.4.0 you decide exactly what gets forwarded.**

Open the app, go to **Settings → SMS Forwarding**, and you'll find three controls.

## 1. The master switch

**SMS Forwarding** on or off. Off means nothing this phone receives is forwarded —
the gateway still sends normally. Leave it on to use the finer controls below.

## 2. Custom senders & short codes

Most of the noise on a SIM comes from **named senders** (like `Vodacom` or `MPESA`)
and **short codes** (like `1234`) — not from real phone numbers. Turn
**Custom senders & short codes** off and the gateway forwards only messages that
come from an actual phone number, quietly dropping OTP and marketing traffic.

Leave it on if you *want* those messages — for example, a service that reads OTPs
on your behalf.

## 3. The blocklist

Need to silence a specific sender while forwarding everything else? Add it to the
**Blocklist** — one number or name per line:

```
+258 84 000 0000
MyBank
1234
```

Anything on the blocklist is never forwarded, even with the switches on. Phone
numbers match regardless of how they're written — `+258…`, `0…`, or the local
form all match the same line, so you don't have to guess the exact format.

## How the three work together

A received message is forwarded only when **all** of these are true:

1. SMS Forwarding is on, **and**
2. the sender is not on your blocklist, **and**
3. it's a real phone number *or* "Custom senders & short codes" is on.

That's it. Grab the
[latest APK](https://github.com/s-mailer/s-mailer-blog/releases/latest/download/s-mailer.apk),
open **Settings**, and tailor forwarding to exactly what your workflow needs.

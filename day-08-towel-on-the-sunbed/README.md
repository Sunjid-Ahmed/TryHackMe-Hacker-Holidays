# Day 08 - Towel on the Sunbed

- **Challenge:** [Towel on the Sunbed](https://tryhackme.com/room/hh-towelonthesunbed-61271709)
- **Category:** Web Exploitation / BurpSuite
- **Difficulty:** Medium
- **Date completed:** 3rd of August 2026

## Summary

Ponzi - Wellness Rewards is a poolside "crypto rewards" app that lets guests claim a daily staking reward, once every 24 hours. The catch: there's a gap between when the claim request hits the server and when it actually records that the reward has been claimed. Wide enough to walk a whale through, as the brief put it.

The goal was simple on paper - create a guest account, work out what stands between you and "Whale" tier, and get past it to reach the Whale Vault and its flag. In practice, that meant exploiting a classic **race condition** on the `/claim` endpoint using Burp Suite.

## Exploitation / Walkthrough

### Step 1: Recon and account setup

Created a guest account (`guest123`) and logged into the Ponzi dashboard. The app had a "Claim Reward" button tied to a 24-hour cooldown, plus a "Whale Vault" section that was locked behind reaching Whale tier.

### Step 2: Intercepting the claim request

With Burp Suite's built-in browser pointed at the target (`MACHINE_IP:3000`), Intercept was switched on and the "Claim Reward" button clicked. *(Note: with Intercept on, the browser can't load the site at all until each request is manually forwarded - Intercept needs to be turned off, or each request allowed through manually, for the page to actually render.)*

This captured the underlying request:

```
POST /claim HTTP/1.1
Host: MACHINE_IP:3000
Content-Length: 0
...
Cookie: connect.sid=...
```

A normal single claim returned a success response with a reward amount and updated balance - but only once. Repeating it manually just returned:

```
HTTP/1.1 429 Too Many Requests
{"error":"Reward already claimed. Please wait before claiming again.","secondsRemaining":86400}
```

Confirming the cooldown was enforced - the question was whether it was enforced *atomically*.

### Step 3: Setting up the race in Repeater

The `/claim` request was sent to Burp Repeater and duplicated into a group of 10 identical tabs (grouped under "Group 1"). The key requirement: this had to be done against a **fresh account/session that had never successfully claimed before** - an account that already had a stored claim would just return `429` on every parallel attempt, since the cooldown was already active before the race even started.

### Step 4: Firing the race condition

With all 10 tabs grouped, Burp's **"Send group (parallel)"** feature was used to fire all 10 `/claim` requests at effectively the same instant. This is the core of the attack: if the server checks "has this user claimed today?" and *then* writes "claimed = true" as two separate steps, sending many requests at once can slip several of them through the check before any of them finish writing the update.

Responses came back as a mix of:

```
HTTP/1.1 200 OK
{"message":"Staking reward claimed successfully.","reward":50,"newBalance":150,"tier":"Whale","priceSnapshot":4.2...}
```

and:

```
HTTP/1.1 429 Too Many Requests
{"error":"Reward already claimed. Please wait before claiming again.","secondsRemaining":86400}
```

One request landed as a genuine `200 OK` win before the server's check caught up - bumping the account balance and, crucially, the account tier to **"Whale"**.

### Step 5: Reaching the vault

With Intercept turned off and the dashboard refreshed, the "Whale" tier unlocked access to the Whale Vault section at the bottom of the page. Clicking **"Open Vault"** revealed the flag.

And voilà, there you have it!

## Flag

![redacted](https://img.shields.io/badge/-REDACTED-000000)

To avoid spoilers, the correct flag will be posted after the event is concluded.

## Lessons Learned

- Race conditions live in the gap between a "check" and a "write" - if those two steps aren't atomic (e.g. not wrapped in a single database transaction or lock), sending enough concurrent requests can slip through the gap.
- The attack only works from a "clean" state - an account that already has a pending cooldown will just get rate-limited on every parallel attempt, since the check already resolves to "already claimed."
- Burp Repeater's "Send group (parallel)" is the tool for this class of bug - it fires grouped requests as close to simultaneously as possible, unlike manually resending requests one by one, which is purely sequential and won't trigger the race.
- Business logic flaws like this don't need any injection or memory corruption - just an understanding of how the app's state changes over time and how to squeeze multiple requests into that window.

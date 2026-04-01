# Brute-forcing a Stay-Logged-In Cookie

## Overview

Some applications let users stay authenticated across browser sessions by setting a persistent cookie. If that cookie is constructed predictably — say, from a username and a hashed password — it becomes possible to forge valid cookies for other users without ever knowing their password directly.

This writeup walks through the process of identifying how such a cookie is built and then using Burp Suite's Intruder to systematically generate valid cookies for a target account.

---

## Identifying the Cookie Structure

Start by logging in with your own credentials and checking the **Stay logged in** option. Once authenticated, open Burp's **Inspector** panel and look at the cookies set on the response.

You'll notice a `stay-logged-in` cookie. Decode it from Base64 and you'll get something like:

```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

The format is clearly `username:something`. The second part is 32 hex characters — that's the fingerprint of an MD5 hash. A reasonable assumption is that it's an MD5 of the user's password.

Quick check to confirm:

```
echo -n "peter" | md5sum
# 51dc30ddc473d43a6011e9ebba6ca770
```

That matches. So the cookie formula is:

```
base64( username + ":" + md5(password) )
```

---

## Setting Up the Attack

Once you understand the structure, log out and intercept the `GET /my-account?id=wiener` request. Send it to **Burp Intruder**.

Intruder should automatically detect the `stay-logged-in` cookie value as a payload position. If not, highlight it manually and mark it.

### Payload Processing Rules

In the **Payloads** tab, under **Payload Processing**, add the following rules in order:

1. **Hash** → MD5
2. **Add prefix** → `carlos:`
3. **Encode** → Base64

This pipeline takes a raw password string, hashes it, prepends the target username, then Base64-encodes the whole thing — producing a valid cookie candidate with every iteration.

### Request Adjustments

Before running the attack against the actual target, make these changes:

- Replace the payload list with the candidate password wordlist
- Change the `id` parameter in the URL from `wiener` to `carlos`
- Make sure the prefix rule uses `carlos:` instead of `wiener:`

### Spotting the Hit

Go to **Settings → Grep - Match** and add `Update email` as a string to flag in responses. That button only appears when you're actually logged in as the account you're viewing, so it's a clean signal.

---

## Running the Attack

Fire off the attack and watch the results. Most responses will come back without the grep match. One of them won't — that's the request where the generated cookie matched Carlos's actual credentials.

The payload from that request is a working `stay-logged-in` cookie for Carlos's account. You can drop it directly into your browser and load `/my-account?id=carlos` to confirm access.

---

## Key Takeaway

The vulnerability here isn't brute force in the traditional sense — it's that the cookie encodes a secret (the password hash) in a fully reversible, guessable way. An attacker who knows the construction and has a wordlist can reconstruct valid cookies offline without ever interacting with the login endpoint.

A safer approach would be to use a cryptographically random session token stored server-side, with no recoverable relationship to the user's credentials.

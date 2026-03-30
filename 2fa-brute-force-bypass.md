# 2FA Bypass via Brute-Force Attack

**Difficulty:** Expert  
**Category:** Authentication  
**Tools Used:** Burp Suite Pro (Intruder + Session Handling Macros)

---

## Overview

This write-up walks through bypassing a two-factor authentication mechanism that lacks brute-force protections. The target account (`carlos:montoya`) has valid credentials, but the 2FA verification code — a 4-digit PIN — stands between us and a successful login.

The core challenge: the application logs you out after two incorrect 2FA attempts, which means any automated attack must re-authenticate before each request. This is solved using Burp Suite's session handling macros paired with Intruder.

---

## Reconnaissance

After logging in as `carlos` and reaching the `/login2` endpoint, a few things become clear:

- The 2FA code is a 4-digit numeric value → **10,000 possible combinations** (0000–9999)
- Entering the wrong code **twice** triggers an automatic logout and redirect back to `/login`
- Each new session generates a fresh verification code, which may invalidate codes already attempted

This last point means we need a strategy that:
1. Re-authenticates before every single code submission
2. Runs requests sequentially (not in parallel) to avoid session collisions

---

## Setting Up the Session Handling Macro

The macro automates the login flow so Intruder always has a valid session before submitting a 2FA guess.

### Step 1 — Open Session Handling Rules

Navigate to **Settings → Sessions → Session Handling Rules** and click **Add**.

### Step 2 — Set URL Scope

In the **Scope** tab, under *URL Scope*, choose **Include all URLs**.  
This ensures the macro fires for every request Intruder sends.

### Step 3 — Add a Macro Action

Back on the **Details** tab, under *Rule Actions*, click **Add → Run a macro**.

### Step 4 — Record the Macro

Click **Add** under *Select macro* to open the **Macro Recorder**.  
Select these three requests in order:

```
GET  /login
POST /login
GET  /login2
```

Click **OK** to open the **Macro Editor**.

### Step 5 — Test the Macro

Click **Test macro** and verify the final response is the 2FA code entry page.  
If it shows up correctly, the macro is working — it's successfully logging in as Carlos and landing on `/login2`.

Click through all the dialogs to save and return to the main Burp window.

---

## Configuring Burp Intruder

### Step 6 — Send the Target Request

Capture the `POST /login2` request and send it to **Intruder**.

### Step 7 — Mark the Payload Position

In the **Positions** tab, clear all auto-detected positions and manually mark the `mfa-code` parameter value as the injection point:

```
mfa-code=§0000§
```

### Step 8 — Configure the Payload

In the **Payloads** panel:

- **Payload type:** Numbers
- **Range:** 0 to 9999
- **Step:** 1
- **Min/Max integer digits:** 4
- **Max fraction digits:** 0

This generates all values from `0000` to `9999`, zero-padded to 4 digits.

### Step 9 — Limit Concurrency

Open the **Resource pool** panel and either create a new pool or configure the existing one with:

```
Maximum concurrent requests: 1
```

Sequential execution is critical here. Running requests in parallel causes session conflicts and breaks the macro's re-authentication flow.

---

## Running the Attack

Start the attack and monitor the **Status** column.

The vast majority of responses will return `200 OK` (invalid code). Keep watching until a `302 Found` appears — that's the successful redirect indicating the correct code was accepted.

> **Note:** Since the 2FA code resets periodically, the correct code might fall in a range Intruder has already passed. If the attack completes without a `302`, simply restart it. The new code will be in the 0000–9999 range and will eventually be hit.

---

## Accessing the Account

Once a `302` response appears:

1. Right-click the request → **Show response in browser**
2. Copy the URL and open it in your browser
3. You're now authenticated as Carlos
4. Navigate to **My Account** to confirm access

---

## Root Cause Analysis

The vulnerability exists because the application:

| Issue | Impact |
|---|---|
| No rate limiting on `/login2` | Allows unlimited code guesses |
| No account lockout after N failures | Brute-force is feasible at scale |
| Short code space (4 digits = 10,000 values) | Exhaustion is trivially fast |
| Logout-on-failure is bypassable | Session macros circumvent the logout defense |

### Remediation

- Implement **exponential backoff** or **account lockout** after 3–5 failed 2FA attempts
- Use **longer codes** (6–8 digits) or **TOTP-based tokens** (time-limited, algorithm-bound)
- Tie the 2FA session to the original login session with a **CSRF token** or **server-side state**
- Consider **IP-based rate limiting** as a secondary layer

---

## Tools & References

- [Burp Suite Pro — Session Handling Rules](https://portswigger.net/burp/documentation/desktop/settings/sessions)
- [Burp Suite Pro — Macros](https://portswigger.net/burp/documentation/desktop/settings/sessions/macros)
- [Turbo Intruder Extension (BApp Store)](https://portswigger.net/bappstore/9abaa233088242e8be252cd4ff534988)
- [PortSwigger Web Security Academy — 2FA Vulnerabilities](https://portswigger.net/web-security/authentication/multi-factor)

---

*Tested on PortSwigger Web Security Academy lab environment. For educational purposes only.*

# Password Reset Broken Logic

**Lab:** Password reset broken logic
**Category:** Authentication
**Difficulty:** Apprentice
**Platform:** PortSwigger Web Security Academy

![Login and reset entry point](images/img2.png)

---

## Overview

This lab demonstrates a **broken password-reset mechanism** in which the reset token fails to serve its security purpose. The reset request carries the token and the target username together, and the application validates only that the submitted token matches the value it expects, without binding that token to a specific user. Because the username is an attacker-controllable parameter in the same request, an attacker can reset the password of any account, including one they do not own.

---

## Objective

- Analyse the password-reset flow using an account under your control.
- Identify how the reset token is validated.
- Abuse the flawed logic to reset the victim's password.
- Log in as the victim and access their "My account" page to solve the lab.

**Your credentials:** `wiener:peter`
**Victim's username:** `carlos`

---

## Vulnerability Classification

| Attribute | Detail |
|---|---|
| Vulnerability | Broken Password-Reset Logic |
| Root Cause | Reset token not bound to a specific user account |
| Impact | Arbitrary account takeover via password reset |

---

## Methodology

### 1. Mapping the Authentication Flow

The application was mapped to identify authentication-related functionality. A login link was located, along with the "forgot password" functionality used to initiate a reset.

![Login and reset entry point](images/img2.png)

### 2. Initiating a Password Reset

Using the account under my control, a password reset was requested. The application sent a reset link to the email inbox I control. The link was clicked and the resulting request was intercepted in Burp Suite for analysis.

![Reset request intercepted](images/img3.png)

### 3. Analysing the Reset Request

Inspection of the intercepted request revealed that the reset **token appeared twice** and that a **username parameter** was present in the same request. The application's logic compared the two token values, and provided they matched, it permitted the password for the supplied username to be reset. Crucially, the token was not tied to any particular account, it was only checked for internal consistency.

![Token and username parameters](images/img4.png)

### 4. Exploiting the Flawed Logic

The request was sent to Repeater. The `username` parameter was changed from my own account to the victim's username, `carlos`, while ensuring the token values remained matched as the application expected. A new password of my choosing was submitted alongside the victim's username.

![Modified request in Repeater](images/img5.png)

### 5. Account Takeover

Because the token was never bound to a specific user, the application accepted the request and reset **Carlos's** password to the value I supplied. Logging in with `carlos` and the new password granted access to the victim's "My account" page, solving the lab.

![Lab solved](images/img6.png)

---

## Root Cause Analysis

The vulnerability lies in the **forgotten-password functionality**, specifically in how the reset token is validated. A password-reset token must act as proof that the requester controls a specific account, typically because it was delivered to that account's registered email. In this implementation, the token is instead treated as a generic value that the application merely checks for consistency within the request. It is not associated with the user whose password is being changed.

Because the target username travels in the same request as an independent, attacker-controllable parameter, the binding between "who proved ownership" and "whose password is changed" is completely absent. The attacker legitimately obtains a valid token for their own account, then swaps the username to the victim's. The application, seeing a matching token and a username, performs the reset without ever asking whether that token was issued for that username. This is a textbook failure to bind a security credential to the identity it is meant to authorise.

---

## Remediation

1. **Bind every reset token to a single account server-side.** The token should be stored against the specific user who requested it, and the reset must change only that user's password. The username should never be accepted as a separate, client-supplied parameter.
2. **Derive the target account from the token, not from the request.** When processing the reset, the server should look up which user the token belongs to and act only on that account.
3. **Use single-use, high-entropy, expiring tokens.** Each token should be unpredictable, valid only once, and time-limited, then invalidated immediately after use.
4. **Do not expose or duplicate the token in client-controllable fields.** The token should not be present in a form in a way that lets the client alter the association between token and account.

---

## Key Takeaway

A password-reset token is only meaningful if it is bound to the exact account it was issued for. This lab shows that validating a token in isolation, without tying it to a specific user, renders the entire mechanism useless. The target account must always be derived from the token on the server side, never accepted as a separate parameter the client can modify. The governing principle is the same one that underpins secure authentication throughout: never trust the client to supply the identity that a security check is meant to protect.

---

## Tools Used

- Burp Suite (Proxy, Repeater)
- Web browser

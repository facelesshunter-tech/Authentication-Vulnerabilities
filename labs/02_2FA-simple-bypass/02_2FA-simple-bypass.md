# 2FA Simple Bypass

**Lab:** 2FA simple bypass
**Category:** Authentication
**Difficulty:** Apprentice
**Platform:** PortSwigger Web Security Academy

---

## Overview

This lab demonstrates a **broken two-factor authentication (2FA)** flaw in which the second authentication factor is not properly enforced on the pages it is meant to protect. The application verifies the password and the 2FA code as separate steps, but the post-authentication account page can be reached directly without ever completing the 2FA step. An attacker who holds a victim's password can therefore skip the 2FA challenge entirely and access the account.

---

## Objective

- Use the victim's known credentials to begin authentication.
- Bypass the 2FA verification step, which is not enforced on the account page.
- Access Carlos's account page to solve the lab.

**Your credentials:** `wiener:peter`

**Victim's credentials:** `carlos:montoya`

---

## Vulnerability Classification

| Attribute | Detail |
|---|---|
| Vulnerability | Broken Two-Factor Authentication (2FA not enforced) |
| Root Cause | Authorization for protected pages not bound to completion of the 2FA step |
| Impact | Full account takeover using only a password, bypassing 2FA |

---

## Methodology

### 1. Accessing the Lab

The lab was launched from the dashboard to reach the target application.

![Lab dashboard](images/img1.png)

### 2. Mapping the Authentication Flow

The application was mapped to identify every authentication-related endpoint. A login link was located and used to authenticate with the provided account `wiener:peter`.

![Login page](images/img2.png)

### 3. Observing the 2FA Step

After submitting valid credentials, the application presented a second step requesting a 2FA verification code.

![2FA prompt](images/img3.png)

### 4. Understanding the Post-Authentication Endpoint

Completing the 2FA step with the `wiener` account led to the account page. The URL of this page was observed to be:

```
/my-account?id=peter
```

This revealed that the account page is a distinct endpoint that takes the username as a parameter, separate from the login and 2FA steps.

![Account page endpoint](images/img4.png)

### 5. Bypassing 2FA for the Victim

The flow was repeated using the victim's credentials `carlos:montoya`. After submitting the password, the application advanced to the 2FA step (`/login2`). Instead of supplying a 2FA code, the browser was navigated directly to the account endpoint, substituting the victim's username:

```
/my-account?id=carlos
```
![Carlos account accessed](images/img5.png)

Because the application did not enforce completion of the 2FA step before serving the account page, this request was granted, and Carlos's account was accessed without ever providing his 2FA code, solving the lab.

![Carlos account accessed](images/img6.png)

---

## Root Cause Analysis

The vulnerability stems from **authentication and authorization being incorrectly decoupled**. The application treats 2FA as a step in a linear flow rather than as a mandatory gate protecting the account resource. It verifies the password, then displays a 2FA page, but the account page itself performs no check that the 2FA step was actually completed for the current session.

In effect, reaching the 2FA page is treated as sufficient progress, while access to the protected account page relies only on knowing the correct endpoint and username. The server never binds the session's authenticated state to the successful completion of the second factor. As a result, the second factor becomes an optional detour rather than an enforced control, and any actor who possesses a valid password can navigate straight past it.

---

## Remediation

1. **Enforce 2FA server-side on every protected resource.** The account page must verify that the current session has fully completed the second-factor step before returning any data, not merely that the password was accepted.
2. **Maintain a strict authentication state machine.** The session should track discrete states — for example, "password verified" versus "fully authenticated" — and grant access to protected pages only in the fully authenticated state.
3. **Never rely on flow order or client navigation for enforcement.** Security must be enforced by server-side checks on each request, since a user can navigate directly to any endpoint.
4. **Bind the account resource to the authenticated user.** The identity served by `/my-account` should derive from the authenticated session, not from a client-supplied `id` parameter.

---

## Key Takeaway

Two-factor authentication provides no protection if it is not enforced on the resources it is meant to guard. This lab illustrates a critical principle: authentication steps must be validated server-side on every protected request, and access must be tied to a session that has genuinely completed all required factors. A second factor that can be skipped by navigating directly to a post-login endpoint is not a control, it is a formality. Enforcement, not presence, is what makes 2FA effective.

---

## Tools Used

- Web browser
- Burp Suite (Proxy, for observing the authentication flow and endpoints)

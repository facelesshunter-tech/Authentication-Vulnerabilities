# Username Enumeration via Subtly Different Responses

**Lab:** Username enumeration via subtly different responses
**Category:** Authentication
**Difficulty:** Practitioner
**Platform:** PortSwigger Web Security Academy

![Objective](images/img1.png)

---

## Overview

This lab demonstrates a **username enumeration** vulnerability in which the difference between a valid and an invalid username is deliberately subtle. Unlike the classic case where the error message changes obviously, here the responses are nearly identical, and the distinguishing signal is a small variation in the response content that is easy to miss. By systematically comparing responses against a known invalid baseline, a valid username can still be isolated, after which the absence of effective brute-force protection allows the password to be recovered.

---

## Objective

- Enumerate a valid username despite near-identical responses.
- Brute-force the identified user's password.
- Authenticate and access the account page to solve the lab.

The lab provides candidate username and password wordlists and contains an account with a predictable username and password.

---

## Vulnerability Classification

| Attribute | Detail |
|---|---|
| Vulnerability | Username Enumeration via Subtly Different Responses |
| Root Cause | Inconsistent authentication responses leaking username validity |
| Impact | Account discovery leading to targeted brute-force and account takeover |

---

## Methodology

### 1. Mapping the Authentication Flow

With traffic proxied, the application was mapped to locate the authentication surface. A login link was identified and opened, presenting a standard login form. A random username and password were submitted first to observe the baseline error message returned for invalid input.

![Login page and baseline error](images/img2.png)

### 2. Initial Enumeration Attempt

The login request was analysed for username enumeration. On first inspection, there was no obvious difference between valid and invalid attempts, no variation in status code, response time, or content length was immediately apparent.

![No obvious difference in responses](images/img10.png)

### 3. Establishing a Known Baseline

To find the subtle signal, a deliberately invalid username was submitted in Repeater to capture the exact invalid-username response. This established a reliable baseline, `Invalid username or password.`, that could be used as the reference for a **negative search**: any response that did not exactly match this baseline would indicate a valid username.

![Baseline invalid response captured](images/img4.png)

### 4. Tooling Constraint and Custom Solution

Performing an efficient negative-match filter across a full wordlist relied on Burp Suite Intruder's result-filtering capability, which is heavily restricted in the free Community edition and gated behind a Professional licence costing several hundred dollars. Rather than pay for the licence, I built a custom Python tool to replicate and extend the required functionality.

> **Intruder — custom Python toolkit.**
> A Burp-free credential-testing tool I developed to replicate Burp Intruder's core attack types and results filtering. It sends login requests from wordlists, saves every response to its own file, and then filters those responses to isolate anomalies.
>
> **Attack types:** Sniper (single input), Pitchfork (paired inputs in lockstep), and Cluster Bomb (every combination) — implemented in Python using `zip` and `itertools.product`.
>
> **Search / filter features:** positive search (files that contain a term), negative search (files that do NOT contain a term — the technique used in this lab), status-code filtering, and response-size filtering, all of which can be combined.
>
> **Other features:** a unified command-line interface with both an interactive menu and subcommands, a layered help/guide system, a local mock server for safe testing, and a clean separated architecture (request engine, search engine, action layer) so the logic can be reused in a future web interface.

![Tool image](images/img5.png)

### 5. Enumerating the Username

Using the tool, a username-enumeration run was performed against the login endpoint with the candidate username wordlist provided by the Academy. Each response was saved for analysis.

![Username enumeration run](images/img6.png)

A **negative search** was then applied, filtering out every response that matched the known invalid baseline `Invalid username or password.`. Only one response remained, revealing the valid username: **`ag`**.

![Valid username identified](images/img7.png)

### 6. Brute-Forcing the Password

Using the identified username `ag`, the tool was run again against the candidate password wordlist to brute-force the password. The successful attempt was distinguished by a **302 redirect** status code, indicating a valid login. The recovered password was **`123123`**.

![Password brute-force with 302 response](images/img8.png)

### 7. Successful Authentication

The recovered credentials `ag:123123` were used to log in, granting access to the account page and confirming the lab was solved.

![Lab solved](images/img9.png)

---

## Root Cause Analysis

The vulnerability arises from **inconsistent authentication responses**. Although the application was designed to return a uniform error, a subtle difference in the response for a valid username, distinct enough to detect through careful comparison, leaks the existence of that account. Enumeration does not require an obvious message difference; any observable deviation in content, length, timing, or status between valid and invalid submissions is sufficient to build an oracle.

This is compounded by **insufficient brute-force protection**. Once a valid username is confirmed, the application permits unlimited password attempts, allowing the password to be recovered directly from the provided wordlist. Individually, a subtle response difference and weak rate limiting are each low-rated issues. Chained together, they create a direct path from anonymous access to full account compromise.

---

## Remediation

1. **Return genuinely identical responses** for all failed authentication attempts, ensuring no difference in body content, response length, status code, or timing between valid and invalid usernames.
2. **Implement rate limiting and account lockout** to prevent automated enumeration and brute-force against confirmed accounts.
3. **Deploy multi-factor authentication** so that a recovered password alone is insufficient to access an account.
4. **Monitor and alert** on high-volume authentication failures indicative of enumeration or brute-force activity.

---

## Key Takeaway

Username enumeration does not depend on an obvious error message. Even a subtle, easily overlooked difference between valid and invalid responses is enough to distinguish real accounts, which is why authentication responses must be identical in every observable dimension. This lab also reinforced a practical reality of the field: when tooling is unavailable or restricted, the ability to build your own is a genuine advantage. Rather than being blocked by a licence paywall, I developed a Python tool that performed the exact negative-search filtering required, and extended it with additional attack types and features, turning a constraint into a stronger capability.

---

## Tools Used

- Intruder (custom Python credential-testing toolkit)
- Burp Suite (Proxy, Repeater)
- PortSwigger candidate username and password wordlists

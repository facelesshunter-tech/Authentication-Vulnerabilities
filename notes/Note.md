# Authentication Vulnerabilities — Complete Notes

> Fundamentals, full OWASP methodology, and modern 2026 attacks.
> **Emmanuel Uwa** · alias *Faceless Hunter*

## 1. What Are Authentication Vulnerabilities

Authentication identifies the user and confirms that they are who they claim to be. Authentication vulnerabilities arise from the insecure implementation of an authentication mechanism in an application. They can result from a design flaw, a logic flaw, a business logic flaw, or a configuration flaw.

**Authentication vs Authorization**
Authentication answers **who are you** (proving identity). Authorization answers **what are you allowed to do** (proving permission). Some attacks in this document, such as the multistage login bypass, cross into authorization because you end up accessing an account you were never permitted to access.

### Common Authentication Mechanisms
- HTML form-based authentication. Example: provide the username and password before being granted access.
- Multi-factor authentication (MFA) mechanisms.
- Windows integrated authentication using NTLM or Kerberos (mostly on internal networks, so you are not constantly re-logging in).

Authentication vulnerability arises from an insecure implementation of the authentication mechanism. It could be the result of:
- Business logic flaw
- Configuration flaw
- Design flaw

---

## 2. Core Authentication Vulnerabilities

### 2.1 Weak Password Requirements

Having no or minimal control over the quality of users' passwords.
- Short password or blank password
- Common dictionary words or names
- The password is the same as the username
- Check for default passwords
- Missing or ineffective MFA

Check out the Consumer Authentication Strength Maturity Model (CASMM).

**How to find and exploit**
- Review the website for any description of the rules based on CASMM.
- If self-registration is possible, attempt to register several accounts with different kinds of weak passwords to discover what rules are in place:
  - Very short or blank password
  - Common dictionary words or names
  - The password is the same as the username
  - Check for the default password
  - Missing or ineffective MFA
- If you control a single account and password change is possible, attempt to change the password to various weak values.

---

### 2.2 Improper Restriction of Authentication Attempts

The application permits brute force or other forms of automated attacks. Applies to:
- Login page
- OTP / MFA
- Change password page

**How to find and exploit**
- Manually submit several bad login attempts for an account you control, up to 10 submissions.
- After 10 failed login attempts, if the application does not return a message about account lockout, attempt to log in correctly. If it works, then there is no lockout mechanism. Run a brute-force attack to enumerate the valid password.
- Tools: Hydra, Burp Intruder, ffuf, etc.
- If the account is locked out, monitor the requests and responses to determine if the lockout mechanism is insecure.

> **Bypassing rate limiting with X-Forwarded-For**
> Many applications enforce lockout or rate limiting per IP address. If the app reads the client IP from the `X-Forwarded-For` header (which the client controls) instead of the real connection IP, an attacker rotates the header value on every request to appear as a new IP each time, defeating lockout entirely. Test by rotating `X-Forwarded-For` across failed logins; if lockout never triggers, the app trusts client-supplied IPs. Also test `X-Forwarded-For: 127.0.0.1` to bypass IP allowlists on admin panels.

---

### 2.3 Verbose Error Message

The application outputs a verbose error message that allows for username enumeration.

> **Clearer example**
> The vulnerability is when the application returns **different** messages for each case — for example, "Username not found" when the username is wrong, versus "Incorrect password" when the username is right but the password is wrong. That difference lets an attacker enumerate valid usernames. The secure version uses one identical generic message for both cases, such as "Invalid username or password."

**How to find and exploit**
- Submit a request with a valid username and an invalid password.
- Submit a request with an invalid username.
- Review both responses for any differences in status code, redirects, information displayed on screen, HTML page source, or even the time taken to process the request.
- If there is a difference, run a brute-force attack to enumerate the list of valid usernames.

> **Timing-based enumeration**
> Even when the message and status code are identical, the response **time** can leak validity. If the backend runs a password-hash comparison only for valid usernames, valid accounts respond measurably slower. Capture and compare response times, not just content.

**NOTE:** Apply this test to all authentication pages.

---

### 2.4 Vulnerable Transmission of Credentials

The application uses an unencrypted HTTP connection to transmit login credentials. You can use Wireshark for demonstration.

**How to find and exploit**
- Perform a successful login while monitoring all traffic in both directions between client and server.
- Look for instances where credentials are submitted in a URL query string or as a cookie, or are transmitted back from the server to the client.
- Attempt to access the application over HTTP and check if there are any redirections to HTTPS.

---

### 2.5 Insecure Forgotten Password Functionality

A design weakness in the forgotten password functionality usually makes it the weakest link that can be used to attack the application's overall authentication logic. For example: using a secret question like "What street did you live on in Sierra Vista?" because with a bit of OSINT, you could find the answer.

**How to find and exploit**
- Identify if the application has any forgotten password functionality.
- If it does, perform a complete walk-through using an account you control while intercepting the requests/responses in a proxy.
- Review the functionality to determine if it allows for username enumeration or brute-force attacks.
- If the application generates an email containing a recovery URL, obtain a number of these URLs and attempt to identify any predictable patterns or sensitive information included in the URL. Also check if the URL is long-lived and does not expire.

---

### 2.6 Defects in Multistage Login Mechanism

Insecure implementation of the MFA/multistage login function. The issue is that the verification code is not associated with the user ID but with the account name.

**How this can be exploited**
Before forwarding the verification code, change the cookie `account` to the account name you want to log in as. If the cookie is not associated with your user ID, it will log you in as the other user.

> **Root cause named**
> The second stage trusts a **client-controlled value** (the account cookie) instead of binding verification to the server-side session established in stage one. The real danger: you complete MFA on **your** account, then swap the cookie, so you bypass the victim's MFA entirely because you never needed theirs. Fix: the second stage must validate against the same server-side session and user ID from stage one, never a user-supplied cookie or parameter.

**How to find and exploit**
- Identify if the application uses a multistage login mechanism.
- If it does, perform a complete walk-through using an account you control while intercepting requests/responses in a proxy.
- Review the functionality to determine if it allows for username enumeration or brute-force attacks.

```http
REQUEST 1
POST /login-steps/first HTTP/1.1
Host: vulnerable-website.com

username=carlos&password=qwerty
```

```http
REQUEST 2
POST /login-steps/second HTTP/1.1
Host: vuln-website.com
Cookie: account=carlos

verification-code=123456
```

---

### 2.7 Insecure Storage of Credentials

Uses plaintext, encoding, reversible encryption, or weakly hashed password storage. If your application gets compromised and your data gets leaked, you will leave your users compromised.

**Storage methods, ranked worst to best**
- **None (plaintext)** — catastrophic; passwords readable directly if breached.
- **Base64 / encoding** — this is encoding, not security; trivially reversible by anyone; provides zero protection.
- **AES256 (encryption)** — wrong choice for passwords because it is reversible; if the key leaks, all passwords are exposed. This is two-way security.
- **MD5** — cryptographically broken and far too fast; common hashes are already cracked and searchable online.
- **SHA256 unsalted** — not broken, but too fast and unsalted; identical passwords produce identical hashes, enabling rainbow-table attacks.
- **bcrypt, scrypt, Argon2** — the correct choice; deliberately slow, memory-hard, and salted. Argon2 is the current gold standard.

> **Key rule:** Passwords must be **hashed, never encrypted, never encoded**. Hashing is one-way; encryption is reversible, which is exactly what you do not want for stored passwords. Every password must have a **unique salt** so identical passwords produce different hashes.

**How to find and exploit**
- If you gain access to the database or backup files during testing, examine how passwords are stored.
- Check whether passwords are plaintext, encoded, reversibly encrypted, or hashed.
- If hashed, determine whether a strong salted algorithm is used (bcrypt/scrypt/Argon2) versus a weak one (MD5 or unsalted SHA256).
- Confirm each password has a unique salt to prevent rainbow-table and precomputation attacks.

---

## 3. Additional OWASP Authentication Tests

These are defined in the OWASP Web Security Testing Guide (WSTG) and complete the full authentication methodology.

### 3.1 Bypassing Authentication Schema (OTG-AUTHN-004)

Getting into a protected area without logging in properly. The app checks credentials at the front door but forgets to lock the back doors.

**1. Direct Page Request (Forced Browsing)** — The app only checks authentication on the login page, not the pages behind it. Log out, then type a protected URL such as `/admin/dashboard` directly. If it loads, the destination was never guarded.

**2. Parameter Modification** — The app decides you are logged in based on a value you can change, e.g. `page.asp?authenticated=no`. Change it to `authenticated=yes`. The same flaw can hide in a cookie or hidden form field.

**3. Session ID Prediction** — Session IDs follow a predictable pattern, so you can guess and hijack another logged-in user's session. Collect several IDs and look for a pattern.

**4. SQL Injection on Login** — Inject SQL into the login form to force the query to return true, e.g. `' OR '1'='1' --` in the username field.

---

### 3.2 Vulnerable Remember-Password Functionality (OTG-AUTHN-005)

The "remember me" feature or browser password saving stores credentials in a retrievable way. Example: a token built by Base64-encoding username and password stored in a cookie — decoded instantly by anyone who obtains it via XSS or a shared computer.

**How to find and exploit**
- Log in with "remember me" checked and examine the cookies.
- If credentials appear in plaintext, Base64, or any easily reversible form, that is the vulnerability.
- Confirm credentials are only sent during login, not attached to every request.

**Fix:** never store actual credentials in the token; store a random, revocable token mapped to a server-side session.

---

### 3.3 Browser Cache Weakness (OTG-AUTHN-006)

After logout, sensitive pages remain in the browser cache or history, retrievable via the back button. Example: view a bank balance, log out on a shared computer, press back, and the balance reappears.

**How to find and exploit**
- Enter sensitive information, log out, then press the browser back button.
- If the sensitive pages still display, the app failed to prevent caching. Also inspect the response headers in a proxy.

**Fix:** sensitive pages must send `Cache-Control: no-cache, no-store`, `Expires: 0`, and `Pragma: no-cache`.

---

### 3.4 Weak Security Question / Answer (OTG-AUTHN-008)

Security questions are often easy to guess, research, or find through OSINT. Example: "What street did you grow up on?" or "Mother's maiden name?" — often discoverable on social media or public records.

**How to find and exploit**
- Check whether security questions are used for recovery or extra security.
- Assess whether answers could be found through social media, public records, or simple guessing.
- Check if users set their own questions, which are often even weaker (e.g. "favorite color").

**Fix:** avoid security questions entirely; if used, treat answers like passwords and never as the only recovery factor.

---

### 3.5 Weaker Authentication in Alternative Channel (OTG-AUTHN-010)

The main website may enforce strong authentication, but the same accounts can often be reached through weaker channels. An attacker attacks the weakest door. Channels to enumerate:
- Mobile-optimized website and mobile app
- Desktop application
- Alternative country or language versions of the site
- Partner websites sharing the same user accounts
- Development, test, staging, and UAT versions of the site
- Call-center operators and interactive voice response (IVR) systems

Example: the main site enforces MFA and lockout, but the mobile API endpoint for the same accounts does not enforce lockout — so an attacker brute-forces passwords there instead.

**How to find and exploit**
- Enumerate every channel that shares the same user accounts.
- For each, list which authentication functions exist and how strong they are (build a comparison grid).
- Look for any channel missing lockout, missing MFA, or with weaker identity checks.

**Fix:** authentication strength must be consistent across every channel that accesses the same accounts.

---

## 4. Modern Authentication Attacks (2026)

> **The core 2026 shift:** Modern systems no longer trust the **person** — they trust the **token**. Once a user authenticates and passes MFA, the server issues a token, and from that moment the token is the identity. Whoever holds it is treated as that user. In 2014 the attacker wanted your password; in 2026 they want your token, because the token is what the system actually trusts.

### 4.1 Token Theft

After login and MFA, the server issues a session token or OAuth bearer token. Whoever holds it is trusted. An attacker who steals the token via XSS or malware imports it into their own browser and gains full access — the password and MFA never come into play.

**How to test / defend**
- Check whether tokens are bound to anything (device, IP, client). If a token works from a completely different device and location with no re-verification, that is the weakness.
- Check token lifetime — a token valid for weeks is far more dangerous than one valid for minutes.

---

### 4.2 Adversary-in-the-Middle (AiTM) Phishing — the MFA Killer

A reverse proxy sits between the victim and the real login portal. The victim sees a genuine login page (forwarded from the real site), enters their password, and completes MFA on their real device. Everything works — but the attacker's proxy captures the resulting session token and replays it. MFA does not fail; the attacker lets the victim complete it and steals the result. Tools such as EvilGinx3, and services like EvilProxy and Tycoon 2FA, have made this accessible to almost anyone.

**Defense**
- **Primary:** phishing-resistant MFA (passkeys / FIDO2 / WebAuthn) — cryptographic domain binding makes the proxy attack impossible in the first place.
- **Secondary:** session-layer continuous evaluation — monitor for a token appearing from a new IP or geography and terminate it; keep token lifetimes short.

---

### 4.3 Consent Phishing (OAuth Abuse)

Needs no password at all. The attacker sends a legitimate OAuth permission screen. The victim clicks Allow, and the attacker receives an access/refresh token to the victim's mailbox, drive, and calendar. Critically, resetting the victim's password does **not** evict the attacker, because they never relied on the password.

> **Important defense distinction:** Passkeys do **not** stop consent phishing, because the victim genuinely authenticates and then willingly grants OAuth permission. The real defenses are: restricting which OAuth apps users can consent to, admin review for high-risk permissions, and monitoring and revoking OAuth grants. This is a common interview trap.

---

### 4.4 Device Code Phishing

Abuses the legitimate OAuth device-code flow (used to log into TVs and input-limited devices). The attacker initiates a device login, gets a code, and sends it to the victim disguised as routine verification. The victim enters it on the real Microsoft/Google page and completes real MFA — but authorizes the attacker's device. Standard MFA does not protect against it.

**Detect / defend**
- Review sign-in logs for device-code grant-type authentications, especially followed by access from unusual IPs or countries.
- Use phishing-resistant MFA and disable the device-code flow where it is not needed.

---

### 4.5 Modern JWT Attacks

JSON Web Tokens are self-contained (header, payload, signature). Key attacks and controls:
- **Algorithm confusion / none attack:** change the alg to `none` or swap RS256 to HS256; if the server accepts it, signature verification is bypassed.
- **Weak secret:** short HMAC secrets can be brute-forced offline.
- **Payload is encoded, not encrypted:** never store sensitive data in a JWT payload.
- **Token binding:** an unbound JWT can be stolen and replayed by anyone — bind tokens to a user/client and validate the signature before processing the payload; check expiry and claims.

---

### 4.6 OAuth 2.0 / SAML Implementation Flaws

OAuth is now everywhere and has its own vulnerability class. Secure implementation requires: exact redirect-URI validation (never loose pattern matching), verifying the state parameter, validating assertion signatures, checking timestamps, and binding tokens to clients.

- **Most testable:** redirect-URI validation. If the app validates the OAuth redirect with loose matching, an attacker crafts a redirect that sends the authorization code to their own server. Exact matching is the fix. This is a common bug-bounty finding.

---

### 4.7 Continuous / Session-Layer Authentication

The old model authenticated once at login and trusted the session forever. Because of token theft and AiTM, the modern model treats authentication as continuous: continuous access evaluation monitors sessions after login, binds sessions to devices, keeps tokens short-lived, and terminates sessions showing anomalous behavior (e.g. a cookie appearing from a new country right after login). Trends also include behavioral biometrics beyond OTP/SMS.

---

## 5. Preventing Authentication Vulnerabilities

- Wherever possible, implement multi-factor authentication — and prefer phishing-resistant MFA (passkeys / FIDO2) over SMS or TOTP.
- A passkey deployment with an SMS fallback is **not** phishing-resistant; the fallback becomes the weakest channel.
- Change all default credentials in production and non-production environments.
- Always use an encrypted channel (HTTPS) when sending user credentials.
- Only POST requests should be used to transmit credentials to the server.
- Stored credentials should be hashed and salted using cryptographically secure algorithms (bcrypt/scrypt/Argon2) — never encrypted or encoded.
- Use identical, generic error messages on the login form, and normalize response length, status code, and timing to prevent enumeration.
- Implement an effective password policy compliant with NIST 800-63b guidelines.
- Use a password-strength checker for real-time feedback, e.g. the zxcvbn JavaScript library.
- Implement robust brute-force protection on all authentication pages, based on the real connection IP or the targeted account — never on the client-controlled X-Forwarded-For header.
- Set `Cache-Control: no-cache, no-store`, `Expires: 0`, `Pragma: no-cache` on sensitive pages.
- Restrict OAuth consent and monitor/revoke grants to defend against consent phishing.
- Keep tokens short-lived, bind them to devices, and apply continuous session evaluation.
- Ensure authentication strength is consistent across every channel (web, mobile, API, call center).
- Audit any verification or validation logic thoroughly to eliminate flaws that allow authentication bypass, and ensure every stage validates against server-side session state, never client-supplied values.

---

## 6. Resources & Tools

### Free resources
- OWASP Web Application Security Testing Guide (Authentication Testing)
- OWASP Application Security Verification Standard (ASVS) — Chapter 2, Authentication Verification Requirements
- PortSwigger Web Security Academy — Authentication Vulnerabilities
- The Web Application Hacker's Handbook

### Web Application Vulnerability Scanners (WAVS)
- Burp Suite
- Acunetix
- w3af
- Wapiti
- Arachni

---

> In 2014 the attacker wanted your password. In 2026 they want your token, because the token is what the system actually trusts. The fundamentals still hold — but if you only learned the fundamentals, you are defending a 2014 threat model in a 2026 world. That gap is exactly where breaches happen.
>
> — **Emmanuel Uwa** (Faceless Hunter)

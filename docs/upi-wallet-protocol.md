# Reversing the UPI Wallet Protocol: From OTP Login to Transaction History

> This is a write-up of an **authorized** Android reverse-engineering study of the client communication protocol used by mainstream Indian UPI wallet apps. App names are referenced only to illustrate cross-platform differences; all endpoints, accounts, and sensitive parameters are redacted. The discussion is limited to the interface model, encryption mechanism, and security controls — it contains no reproducible attack code and targets no real account or service.

**Platforms covered:** Amazon Pay, MobiKwik, PhonePe, BharatPe Business, Airtel, FreeCharge, Paytm
**Objective:** Abstract the interface model, parameter conventions, encryption mechanism, and security measures of these UPI wallet apps across three core functions — **OTP login, UPI ID (VPA) lookup, and transaction history retrieval** — for use in security research, risk-control modeling, and compliance audit.

---

## 1. Overall Architecture

### 1.1 Transport

All platforms run over **HTTPS (TLS)**, with some enabling **HTTP/2** to reduce handshake overhead. Request bodies are predominantly **JSON**; a few legacy endpoints (e.g. Amazon Pay) still use the older **server-rendered HTML + client-side parsing** pattern.

### 1.2 Session Lifecycle

A UPI wallet session follows a strict linear pipeline with **hard state dependencies** — each step's output is the next step's input:

```
Device init / registration
        ↓
Fetch encryption public key (some platforms)
        ↓
Request OTP send (phone number entered)
        ↓
Verify OTP → obtain Auth Token / Cookie / Session
        ↓
Query UPI ID (requires logged-in Token)
        ↓
Query transaction list (requires Token, paginated)
        ↓
Query transaction detail (optional)
        ↓
Token refresh / logout
```

An OTP request that omits the prerequisite Cookie is rejected outright; a Token that did not pass OTP verification cannot reach the UPI ID or history endpoints. This "progressive unlock" is itself a layer of defense-in-depth.

---

## 2. The OTP Login Flow

OTP (one-time password) login is the standard authentication method for UPI wallets, split into a **send** step and a **verify** step:

- **Step 1 — Request OTP send:** the client submits a phone number; the server sends a 4–6 digit code to that number.
- **Step 2 — Verify OTP:** the client submits the code; on success the server returns an auth credential (Token / Cookie) used as the identity for all subsequent calls.

Some platforms (e.g. Paytm) layer a multi-step **OAuth2 authorization** on top of OTP verification (state token → authorization code → access token), ultimately producing a Bearer Token valid for roughly **one month**.

### 2.1 Prerequisite Steps

Before requesting an OTP, most platforms require one or more initialization actions. This means a "bare" OTP request is almost impossible to succeed against a modern UPI wallet:

| Step | Description | Typical params |
|---|---|---|
| Device registration / config | Report device info, receive a device-level initial Token / Session | device ID, OS version, model, app version |
| Fetch encryption public key | Pull an RSA public key (~36h validity) for later body encryption | no input; returns key string + TTL |
| Obtain initial Cookie / CSRF token | Initialize session context | delivered via `Set-Cookie` |
| Initialize Session ID | Create the login session | phone number, device ID, login flow type |

### 2.2 Request-OTP Endpoint

- **Method:** `POST`
- **Auth state:** anonymous (no login)

Beyond business fields, the Header layer carries extensive device and crypto metadata:

| Location | Parameter | Notes |
|---|---|---|
| Header | `Content-Type` | `application/json` |
| Header | `User-Agent` | client identifier (app version, OS, model) |
| Header | Device ID | platform-specific: `X-Device-ID` / `x-bsy-did` / `x-id` |
| Header | App version | `X-App-Ver` / `x-bsy-appvn` / `x-app-version` |
| Header | Encryption public key | some platforms place the RSA key in a header (e.g. `pke`) |
| Header | Encryption key handle | AES key wrapped with RSA (e.g. `cske`), for server-side body decryption |
| Header | CSRF token | from a prerequisite step; anti-CSRF |
| Body | Phone number | plaintext or symmetrically encrypted |
| Body | Device ID | matches the Header |
| Body | Nonce | `nonceToken` / `tid`; anti-replay |

**Key success-response fields:**

| Field | Meaning |
|---|---|
| `otpId` / `nid` / `sessionId` | unique ID for this OTP; must be echoed back at verification |
| `otpLength` | digit count (4 or 6) |
| Status flag | platform-specific: `success:true` / `status:"SUCCESS"` |

**Common errors:** phone number not found / malformed, rate-limited (too frequent), device not authorized.

### 2.3 Verify-OTP Endpoint

- **Method:** `POST`
- **Auth state:** anonymous

The client submits the user-entered 4–6 digit code plus the correlation identifier (`otpId` / `sessionId` / `nonceToken`) returned by the send step. On success the server issues an **auth credential** — and the credential's concrete shape is where platforms diverge most:

| Token form | Description | Platform |
|---|---|---|
| HTTP Cookie (e.g. `app_fc`) | stored in `Set-Cookie`, auto-attached afterwards | FreeCharge |
| Header-field concatenation | `hashid + "." + token` combined into `Authorization` | MobiKwik |
| `accessToken` in body | Bearer Token; manually injected into `Authorization` | Paytm, PhonePe |
| Multi-field Session Cookie | `session-token`, `at-acbin`, etc. jointly maintain the session | Amazon Pay |
| Dynamic + static dual Token | dynamic Token refreshed per request, static Token held long-term | Airtel |

**Common verify-OTP failure cases:**
- Wrong code (remaining attempts decrement)
- Expired code (typically valid ~**3–5 minutes**)
- Max attempts exceeded (usually **5 tries → 24h lockout**)

---

## 3. The Encryption Scheme

UPI wallets generally use a **hybrid RSA + AES** scheme that balances security and performance:

- **Asymmetric layer (RSA):** the client fetches the server's RSA public key (~36h validity) and uses it to wrap a one-time AES key.
- **Symmetric layer (AES):** the actual business body (e.g. phone number) is encrypted with AES; the AES key itself is RSA-wrapped and placed in a Header (`cske`) so the server can recover it with its private key.

This mirrors the "double-layer" idea of running application-layer encryption inside TLS: even if the transport layer is observed by a man-in-the-middle (e.g. a debug proxy), the application-layer ciphertext still protects sensitive fields.

Three auxiliary identifiers each serve a distinct purpose:

| Identifier | Purpose |
|---|---|
| `nonceToken` / `tid` | **Anti-replay** — a request cannot be submitted twice |
| CSRF token | **Anti-CSRF** |
| Device fingerprint (ID / model / OS / app version) | **Device binding & risk control** |

---

## 4. UPI ID Lookup and Transaction History

Once the auth credential is in hand, two core business paths open up:

1. **UPI ID (VPA) lookup:** requires a logged-in Token; returns the user's registered Virtual Payment Address (e.g. `xxxx@bank`) and binding status. Some platforms attach this directly in the OTP-verify response.
2. **Transaction history:** also Token-gated, supports paginated list retrieval, with optional single-transaction detail.

These two endpoints directly expose a user's financial trail and are the focus of UPI wallet risk control and privacy protection — typically carrying additional device checks and rate limits.

---

## 5. Security Controls Summary

Across the generic protocol, a UPI wallet's security design resolves into five defensive lines:

| Defensive line | Mechanism |
|---|---|
| Transport security | HTTPS / TLS, some HTTP/2 |
| Application-layer crypto | RSA + AES hybrid, short public-key rotation (~36h) |
| Authentication | two-step OTP + multi-step OAuth (Paytm) |
| Session binding | device fingerprint, CSRF, Session ID |
| Risk / rate limiting | OTP throttling, attempt lockout (5 / 24h), anti-replay nonce |

---

## 6. Methodology Notes

UPI wallet apps look simple on the surface, but their communication protocol is a composite system of **hard state dependency + multi-layer encryption + multi-form credentials**. Platforms differ in token shape, prerequisite steps, and crypto details, yet the underlying model is highly consistent — which is exactly why a "generic protocol" can be abstracted at all.

A recommended analysis path for reversing such apps:

1. **Establish session context first.** Always capture from a cold start and record the full prerequisite chain — device registration → public-key fetch → cookie acquisition — or later requests won't replay.
2. **Identify the crypto layer.** Locate the client-side RSA public-key cache and AES key-generation logic; they usually concentrate in the network layer or a dedicated crypto class.
3. **Track token lifecycles.** Distinguish short-lived from long-lived tokens, and note their different injection points across Header / Cookie / Body.
4. **Watch for risk signals.** Rate limiting, attempt lockout, and device-fingerprint checks are "implicit constraints" outside the protocol proper, and they often define the feasibility boundary of any automation.

Understanding this model is directly useful for **mobile-payment security research, client-side reverse engineering, and risk-control rule construction**.

---

<sub>This document is for technical demonstration and authorized research only. It provides no tools or methods for unauthorized access to real payment services.</sub>

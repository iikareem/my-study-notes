---
tags:
  - auth
  - security
---

# JWT vs PASETO: A Complete Guide

## Table of Contents

1. [What is PASETO and What Problem Does It Solve](#1-what-is-paseto-and-what-problem-does-it-solve)
2. [PASETO Structure](#2-paseto-structure)
3. [PASETO v4 Crypto Details](#3-paseto-v4-crypto-details)
4. [Claims / Payload](#4-claims--payload)
5. [Practical Usage Guide](#5-practical-usage-guide)
6. [The "none" Algorithm Attack](#6-the-none-algorithm-attack)
7. [Algorithm Confusion Attack (RS256 → HS256)](#7-algorithm-confusion-attack-rs256--hs256)
8. [Who Is Actually Vulnerable: Private Key vs Public Key Holder](#8-who-is-actually-vulnerable-private-key-vs-public-key-holder)
9. [Final Takeaway](#9-final-takeaway)

---

## 1. What is PASETO and What Problem Does It Solve

**PASETO** = **P**latform-**A**gnostic **SE**curity **TO**kens.

Starting from JWT knowledge: JWT is conceptually solid but **too flexible**. It lets the algorithm be specified _inside_ the token itself via the `alg` header:

```json
{ "alg": "HS256", "typ": "JWT" }
```

This flexibility ("crypto agility") has caused real-world vulnerabilities:

- The **`none` algorithm attack** — a token can claim to be unsigned, and a careless verifier just believes it.
- **Algorithm confusion attacks** (RS256 → HS256) — a verifier can be tricked into treating an RSA public key as an HMAC secret.
- **Weak library defaults** — historically, many libraries shipped insecure defaults, requiring developers to correctly lock down allowed algorithms.

### PASETO's guiding principle

> **No algorithm agility.** You pick a _version_ of PASETO, and that version has exactly ONE way to encrypt/sign tokens. Nothing to misconfigure.

Instead of "here are 12 algorithms, configure the right one," PASETO says: "version 4 uses XChaCha20-Poly1305 or Ed25519, period."

---

## 2. PASETO Structure

```
v4.local.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIn0...
```

Format: `version . purpose . payload . [optional footer]`

### Version

`v1`, `v2`, `v3`, `v4` — defines the crypto primitives used. **Use `v4`** unless you have a specific compliance reason (e.g., v3 for FIPS-approved algorithms).

### Purpose

Only two options — no negotiation possible:

|Purpose|Meaning|JWT Equivalent|
|---|---|---|
|`local`|Symmetric **encryption** — payload is encrypted + authenticated|JWE (encrypted JWT)|
|`public`|Asymmetric **signing** — payload is plaintext/base64, but signed|JWS (standard signed JWT)|

This is an important mental shift from JWT: a standard JWT (signed only) is **readable by anyone**, even though people often mistakenly assume it's "protected." PASETO makes this explicit through the `local` vs `public` distinction.

### Payload

Base64url-encoded ciphertext (`local`) or JSON claims + signature (`public`).

### Footer (optional)

Extra metadata (e.g., key ID) — authenticated but not encrypted. Useful for key rotation.

---

## 3. PASETO v4 Crypto Details

|Purpose|Algorithm|
|---|---|
|`v4.local`|XChaCha20-Poly1305 (encryption + authentication)|
|`v4.public`|Ed25519 (digital signature)|

You never _choose_ these — version + purpose locks them in automatically.

---

## 4. Claims / Payload

PASETO reuses JWT's registered claim names, so existing knowledge transfers:

```json
{
  "sub": "user-1234",
  "exp": "2026-07-01T12:00:00+00:00",
  "iat": "2026-07-01T11:00:00+00:00",
  "aud": "my-app",
  "iss": "my-auth-server"
}
```

**Key difference:** PASETO requires **ISO 8601 timestamps** for `exp`/`iat`/`nbf`, unlike JWT's Unix epoch integers. Watch for this when migrating.

---

## 5. Practical Usage Guide

### Node.js (`paseto` package)

```bash
npm install paseto
```

**`v4.local` (encrypted) token:**

```javascript
const { V4 } = require('paseto');

async function run() {
  const key = await V4.generateKey('local'); // store securely!

  const token = await V4.encrypt(
    {
      sub: 'user-1234',
      role: 'admin',
      exp: new Date(Date.now() + 15 * 60 * 1000).toISOString()
    },
    key
  );

  const payload = await V4.decrypt(token, key);
}

run();
```

**`v4.public` (signed) token:**

```javascript
const { V4 } = require('paseto');

async function run() {
  const { publicKey, secretKey } = await V4.generateKey('public');

  const token = await V4.sign({ sub: 'user-1234', role: 'admin' }, secretKey);

  const payload = await V4.verify(token, publicKey);
}

run();
```

### Other language libraries

- **Python**: `pyseto` or `paseto`
- **Go**: `github.com/o1egl/paseto` or `aidantwoods.com/go-paseto`
- **PHP**: `paragonie/paseto` (by PASETO's own creator)
- **Rust**: `rusty_paseto`

### `local` vs `public` — when to use which

- **`local`**: same service issues and verifies tokens (e.g., your own backend session tokens). Payload stays confidential.
- **`public`**: other services need to verify without being trusted to issue (e.g., microservices verifying tokens from a central auth service). Mirrors RS256/ES256 JWT usage.

### Comparison Table

|Aspect|JWT|PASETO|
|---|---|---|
|Algorithm selection|Developer chooses — many pitfalls|Fixed per version — zero choice|
|Confidentiality|Needs separate JWE spec, rarely used correctly|`local` purpose built-in|
|"none" algorithm attack|Possible if misconfigured|Impossible by design|
|Algorithm confusion attack|Possible (RS256/HS256 confusion)|Impossible — no `alg` field exists|
|Ecosystem maturity|Extremely mature, everywhere|Smaller but growing|
|Timestamp format|Unix epoch (numeric)|ISO 8601 (string)|
|Library support|Nearly universal|Check before committing|

---

## 6. The "none" Algorithm Attack

### Background

The JWT spec technically allows `"alg": "none"` — meaning "no signature, don't verify." Intended for niche internal use cases, but many libraries implemented it too permissively.

### How it works

Legitimate token:

```
Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "user123", "role": "user"}
Signature: <HMAC signature>
```

Attacker steps:

1. Decodes header/payload (trivial — it's just base64, not encrypted).
2. Changes `"role": "user"` → `"role": "admin"`.
3. Changes `"alg": "HS256"` → `"alg": "none"`.
4. Removes the signature entirely.

Resulting forged token:

```
Header:  {"alg": "none", "typ": "JWT"}
Payload: {"sub": "user123", "role": "admin"}
Signature: (empty)
```

### Why it worked

Vulnerable code trusted the token's own header to decide whether verification was even needed:

```javascript
const decoded = jwt.decode(token); // no algorithm check!
```

### The fix

Explicitly allow-list acceptable algorithms on the verifier side:

```javascript
jwt.verify(token, secretKey, { algorithms: ['HS256'] });
```

This is a **developer-discipline fix** — it depends on remembering to configure it correctly every time.

---

## 7. Algorithm Confusion Attack (RS256 → HS256)

### Background: two signing modes

- **HS256 (HMAC)** — symmetric. One secret key both signs and verifies.
- **RS256 (RSA)** — asymmetric. A **private key** signs; a **public key** (safe to share openly) verifies.

### The vulnerable setup

A server expects RS256 tokens. It has a public key available (e.g., published at `/.well-known/jwks.json`). Vulnerable code:

```javascript
// VULNERABLE: doesn't pin the expected algorithm
const decoded = jwt.verify(token, publicKey);
```

### How the attack works

1. Attacker obtains the server's **public key** (trivial — it's meant to be public).
2. Crafts a forged token:
    
    ```
    Header:  {"alg": "HS256", "typ": "JWT"}Payload: {"sub": "user123", "role": "admin"}
    ```
    
3. Signs it with **HMAC-SHA256**, using the **RSA public key string itself as the HMAC secret**.
4. Server sees `"alg": "HS256"` in the header and — because the code doesn't pin the algorithm — switches to HMAC verification mode, using the public key as the HMAC secret.
5. Since the attacker used that exact same string to sign, **the signature checks out**.

### Why it's dangerous

It inverts the trust model. RS256's public key was never meant to be secret — but if the verifier doesn't pin the algorithm, that same public key can be abused as a symmetric HMAC secret, letting an attacker forge tokens without ever touching the real private key.

### The fix

Pin the expected algorithm explicitly:

```javascript
jwt.verify(token, publicKey, { algorithms: ['RS256'] }); // reject anything else
```

### The common thread with the `none` attack

> **The verifier trusted the token to tell it how to verify itself.**

The `alg` field is attacker-controlled input (headers are just base64, freely editable). Using untrusted input to decide _how much to trust that same input_ is a classic security anti-pattern — similar in spirit to SQL injection or XXE attacks.

---

## 8. Who Is Actually Vulnerable: Private Key vs Public Key Holder

**Important clarification:** the algorithm confusion attack targets services that verify using the **public** key — not the private key holder.

|Role|Has private key?|Has public key?|Can create valid RS256 tokens?|Can be tricked into accepting forged tokens?|
|---|---|---|---|---|
|Auth server (issuer)|Yes|Yes|Yes (legitimately)|N/A — doesn't verify others' tokens|
|Downstream service (verifier)|No|Yes|No (can't forge real RS256)|**Yes, if misconfigured** — via the HS256 swap|
|Attacker|No|Yes (it's public!)|No|N/A — is the one exploiting it|

### Why this makes sense

The whole point of asymmetric signing is: _"anyone can verify, but only the private key holder can create."_ Public keys are **meant** to be shared freely — that's not a leak, that's the design.

The vulnerability isn't "the public key leaked." It's that **the verifier's code is willing to reinterpret that same public key as a symmetric HMAC secret** if the token's header asks it to. That reinterpretation logic is the flaw — not the public key's public-ness.

The private key and its holder (the auth server) are never touched or compromised. This is what makes the attack clever: it bypasses strong crypto entirely by exploiting a **logic bug in verification code**, not by breaking RSA itself.

---

## 9. Final Takeaway

**JWT the cryptography is secure.** RS256, HS256, ES256 are all legitimate, well-vetted algorithms — there's no flaw in RSA, HMAC, or ECDSA themselves.

**JWT the specification/ecosystem is what's risky**, because:

1. It gives implementers _choices_ (which algorithm? which library options?)
2. Those choices have _insecure paths available_ (`none`, algorithm switching, weak defaults)
3. Security correctness depends on **every developer, every time, configuring it exactly right**

This reflects the difference between a system being **theoretically secure** vs. **secure by default / hard to misuse**.

### PASETO's actual innovation

- JWT: _"Please remember to allow-list your algorithms"_ — a runtime check developers must remember to add.
- PASETO: _"There is no algorithm field to check, because version+purpose makes it structurally impossible to specify a different one"_ — a design-time guarantee, nothing to forget.

This reflects the security design principle of **"secure by design"** / **"making insecure states unrepresentable."** Instead of relying on developers to avoid the footgun, PASETO removes the footgun from existence.

### The trade-off

This safety costs **flexibility**. JWT's negotiability is a double-edged sword: it enables misconfiguration, but it also lets JWT slot into many existing systems and standards (OAuth2, OIDC, FIPS compliance) that expect JWT specifically. PASETO deliberately traded that flexibility for safety — which is why it hasn't fully displaced JWT industry-wide, but is a strong choice for new systems where you control both the issuer and verifier.

> **Summary:** JWT is cryptographically sound but configuration-fragile. PASETO exists specifically to remove that fragility by removing the configuration surface entirely.
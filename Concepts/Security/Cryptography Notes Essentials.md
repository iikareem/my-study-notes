# Cryptography Notes: Essentials

## Part 1: Foundations

### The 4 security goals

| Goal                | Meaning                          |
| ------------------- | -------------------------------- |
| **Confidentiality** | No one else can read it          |
| **Integrity**       | It wasn't modified               |
| **Authenticity**    | It came from who it claims       |
| **Non-repudiation** | The sender can't deny sending it |

### Which primitive gives what

|Primitive|Confidentiality|Integrity|Authenticity|Non-repudiation|
|---|:-:|:-:|:-:|:-:|
|Symmetric encryption (GCM)|✅|✅|✅ (shared key)|❌|
|Asymmetric encryption|✅|❌|❌|❌|
|Hash|❌|⚠️ accidental only|❌|❌|
|HMAC|❌|✅|✅ (shared key)|❌|
|Digital signature|❌|✅|✅|✅|

### Vocabulary

- **Plaintext / ciphertext**: readable data / encrypted data
- **Key**: the secret (or key pair) that controls the operation
- **Nonce / IV**: a unique value per encryption, never reused with the same key
- **Salt**: a random value added to a password before hashing
- **Digest**: the fixed-size output of a hash

---

## Part 2: Encryption

### 2.1 Symmetric encryption

**The same key encrypts and decrypts.**

```
plaintext --[key]--> ciphertext --[same key]--> plaintext
```

- **Algorithms:** AES-256-GCM (standard), ChaCha20-Poly1305
- **Pros:** very fast, good for large data
- **Cons:** key distribution problem (how do both sides get the key securely?)
- **Use cases:** DB field encryption, files, disk encryption, bulk data in TLS

**Must-know:**

- Use an **authenticated mode** (GCM). Plain AES-CBC has no tamper protection.
- **Never reuse a nonce** with the same key.

### 2.2 Asymmetric encryption

**Key pair: the public key encrypts, the private key decrypts.**

```
plaintext --[public key]--> ciphertext --[private key]--> plaintext
```

- **Algorithms:** RSA (with OAEP padding), ECC (ECDH for key exchange)
- **Pros:** no pre-shared secret, and the public key can be shared openly
- **Cons:** slow, and limited in how much data it can encrypt directly
- **Use cases:** key exchange, encrypting small secrets

### 2.3 Hybrid encryption

In practice you combine both:

1. **Asymmetric** to securely agree on or exchange a key
2. **Symmetric** to encrypt the actual data with that key

This is how HTTPS works. You get the convenience of asymmetric and the speed of symmetric.

---

## Part 3: Digital Signatures

It is related to asymmetric encryption because it uses the **same key pair in reverse**: the **private key signs, the public key verifies**.

```
Sign:    hash(message) --[private key]--> signature
Verify:  check signature with public key against hash(message)
```

- **Gives you:** integrity + authenticity + **non-repudiation**
- Only the private key holder can sign, but **anyone** can verify
- **Algorithms:** RSA-PSS, ECDSA, Ed25519
- **Use cases:** TLS certificates, code signing, software updates, JWT `RS256`/`ES256`

> A signature signs the **hash** of the message (covered next in Part 4), not the whole message.

### Encryption vs signing

||Encryption|Signing|
|---|---|---|
|Public key is used to|Encrypt|Verify|
|Private key is used to|Decrypt|Sign|
|Goal|Confidentiality|Authenticity + integrity|

**Must-know:** signing does **not** hide the data. The message stays readable. Encryption and signing are separate operations, and you can do both.

---

## Part 4: Hashing

**A one-way function that maps any input to a fixed-size output (digest).**

```
"hello" --> SHA-256 --> 2cf24dba5fb0a30e...
```

### Properties

- **Deterministic:** same input, same output
- **One-way:** you can't get the input back from the digest
- **Avalanche effect:** a tiny input change gives a completely different output
- **Collision-resistant:** practically impossible to find two inputs with the same digest

### Algorithms

SHA-256, SHA-3, BLAKE2. **Avoid MD5 and SHA-1** (broken).

### Use cases

- File integrity and checksums (verify a download)
- Content addressing (Git) and deduplication
- Building block inside **signatures** and **HMAC**

### Limitation

There is **no key**, so anyone can compute it. An attacker can modify a message and recompute the hash. A plain hash only protects against accidental corruption. This is why HMAC exists.

---

## Part 5: Password Hashing

**Use a dedicated password hash. Not plain SHA-256, and not HMAC alone.**

SHA-256 is built to be fast, so an attacker with a leaked DB can try billions of guesses per second.

|Required property|Why|
|---|---|
|**Slow (tunable cost)**|Makes each guess expensive|
|**Salted (unique per user)**|Defeats rainbow tables, so identical passwords give different hashes|
|**Memory-hard**|Makes GPU/ASIC cracking costly|

### Algorithms

1. **Argon2id**: first choice for new systems
2. **scrypt**: good alternative
3. **bcrypt**: widely supported, acceptable (72-byte input limit)

### Pepper (optional extra)

A server-side secret, stored outside the DB (env/KMS), applied before hashing:

```
stored = Argon2id( HMAC-SHA256(pepper, password) )
```

If only the DB leaks, the attacker still lacks the pepper. It is defense in depth, not a replacement.

### Node.js

```js
const argon2 = require('argon2');

const hash = await argon2.hash(password);        // salt + params embedded
const ok = await argon2.verify(hash, password);  // constant-time compare
```

**Must-know:** never use MD5, SHA-1, or unsalted SHA-256 for passwords. Never write your own scheme. Rehash on login when parameters are outdated.

---

## Part 6: HMAC

**A hash combined with a secret key.**

```
Hash:  SHA-256(message)           → anyone can compute it
HMAC:  HMAC-SHA256(key, message)  → only key holders can compute it
```

- Both parties share the **same secret**
- Gives **integrity + authenticity**
- Does **not** give non-repudiation, because both sides can generate the same tag
- **Use cases:** webhook verification (Stripe, GitHub), API request signing, JWT `HS256`, signed cookies

### Why it beats a plain hash

```
Hash only: attacker changes "pay $10" to "pay $1000" and recomputes the hash → still verifies
HMAC:      attacker can't produce a valid tag without the secret → verification fails
```

### Node.js

```js
const crypto = require('crypto');

const sig = crypto.createHmac('sha256', secret).update(body).digest('hex');

// Constant-time comparison (avoids timing attacks)
const valid = crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(receivedSig));
```

**Must-know:**

- Always compare with `timingSafeEqual`, not `===`.
- HMAC is **not** `hash(key + message)`. That is vulnerable to **length extension attacks**. HMAC's nested construction prevents it.

---

## Part 7: Comparisons

### Hash vs HMAC

||Hash|HMAC|
|---|---|---|
|Uses a key|No|Yes|
|Detects accidental change|Yes|Yes|
|Detects malicious tampering|No|**Yes**|

### HMAC vs Digital Signature

||HMAC|Digital Signature|
|---|---|---|
|Key type|Shared secret|Private/public pair|
|Who can create it|Anyone with the secret|Only the private key holder|
|Who can verify it|Anyone with the secret|Anyone with the public key|
|Non-repudiation|No|**Yes**|
|Speed|Very fast|Slower|
|Best for|Two trusted parties|Public or multi-party verification|

### Symmetric vs Asymmetric

||Symmetric|Asymmetric|
|---|---|---|
|Keys|1 shared|Public + private pair|
|Speed|Fast|Slow|
|Main problem|Key distribution|Performance, size limits|
|Typical role|Encrypt bulk data|Exchange keys, sign|

### Master table

|Primitive|Key|Reversible|Main purpose|
|---|---|---|---|
|Symmetric encryption|1 shared|Yes|Confidentiality|
|Asymmetric encryption|Key pair|Yes|Confidentiality / key exchange|
|Hash|None|**No**|Integrity fingerprint|
|HMAC|1 shared|No|Integrity + authenticity|
|Digital signature|Key pair|No|Integrity + authenticity + non-repudiation|

---

## Part 8: How They Work Together

### TLS (HTTPS), simplified

1. Server sends a **certificate**, which is a **digital signature** by a CA over the server's public key
2. Client verifies the signature and trusts the public key
3. Both sides use **asymmetric** key exchange (ECDH) to agree on a shared key
4. The session uses **symmetric** encryption (AES-GCM) for data and integrity
5. **Hashes** are used inside signatures and key derivation

### JWT: HS256 vs RS256

||HS256 (HMAC)|RS256 / ES256 (signature)|
|---|---|---|
|Key|Shared secret|Private signs, public verifies|
|Use when|One service issues and verifies|Many services or third parties verify|
|Risk|Every verifier can also forge tokens|Private key stays only on the issuer|

For microservices or a BFF verifying tokens from an auth service, prefer **RS256/ES256**.

### Webhook verification

Sender computes `HMAC(secret, body)` and sends it in a header. Receiver recomputes it and compares in constant time.

---

## Part 9: Decision Cheat Sheet

|I need to...|Use|
|---|---|
|Encrypt large data, files, DB fields|Symmetric (AES-256-GCM)|
|Share a key securely with someone|Asymmetric key exchange (ECDH/RSA)|
|Verify a file wasn't corrupted|Hash (SHA-256)|
|Store passwords|Argon2id / bcrypt|
|Verify a webhook from a trusted partner|HMAC|
|Let many services verify who issued a token|Digital signature (RS256/ES256)|
|Prove something came from one specific party|Digital signature|

---

## Part 10: Common Mistakes and Interview Questions

### Common mistakes

- Using MD5/SHA-1 for anything security-related
- Hashing passwords with plain or unsalted SHA-256
- Comparing HMACs with `===` instead of constant-time comparison
- Reusing a nonce/IV with the same key in AES-GCM
- Thinking a **hash is encryption** (it can't be decrypted)
- Thinking a **signature hides data** (it doesn't)
- Using asymmetric encryption directly on large payloads
- Rolling your own crypto

### Interview questions (short answers)

**1. Hash vs encryption?**  
Encryption is reversible with a key. Hashing is one-way with no key.

**2. Hash vs HMAC?**  
HMAC adds a secret key, so an attacker can't forge a valid tag. A plain hash can be recomputed by anyone.

**3. HMAC vs digital signature?**  
HMAC uses a shared secret, so both sides can create it and there is no non-repudiation. A signature uses a key pair, so only the private key holder can sign and anyone can verify.

**4. Why not SHA-256 for passwords?**  
It's too fast. Use slow, salted, memory-hard algorithms like Argon2id or bcrypt.

**5. What is a salt, and what is a pepper?**  
A salt is random and unique per user, stored with the hash. A pepper is a secret stored outside the DB and shared across users.

**6. Why is asymmetric encryption not used for all data?**  
It's slow and has size limits. It's used to exchange a symmetric key instead (hybrid).

**7. How does HTTPS combine all of these?**  
Signature (certificate) for authenticity, asymmetric key exchange to agree on a key, symmetric AES-GCM for the data, hashes inside all of it.

**8. HS256 vs RS256?**  
HS256 uses a shared secret, so every verifier can also forge tokens. RS256 uses a private key to sign and a public key to verify, which is better for multiple services.

**9. Does a digital signature encrypt the message?**  
No. It proves origin and integrity only. The message stays readable.

**10. Why use constant-time comparison?**  
A normal `===` can leak how many bytes matched through timing differences, which lets an attacker guess a valid MAC byte by byte.
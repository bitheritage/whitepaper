# BitHeritage: Secure Secret Sharing Inheritance Protocol

**Version 0.1 — Initial Draft**

---

## Abstract

BitHeritage is an open-source protocol and platform for securely sharing sensitive secrets — such as cryptocurrency seed
phrases, passwords, or any other important and highly secure information — between two parties with a time-delayed
inheritance mechanism. The system guarantees that secrets remain encrypted at every stage, that neither the server nor
any third party can access plain data, and that the original owner retains full control over shared secrets until a
configurable grace period expires. Authentication is handled exclusively through WebAuthn passkeys, eliminating
passwords entirely and upholding a zero-knowledge principle where the platform never learns the user's identity beyond a
self-chosen tag name.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Design Principles](#design-principles)
3. [User Flow](#user-flow)
4. [Grace Period and Access Control](#grace-period-and-access-control)
5. [Trust Model and Threat Analysis](#trust-model-and-threat-analysis)
6. [Roadmap](#roadmap)
7. [Conclusion](#conclusion)

---

## Problem Statement

Millions of people hold digital secrets — cryptocurrency seed phrases, master passwords, encryption keys — that would be
permanently lost if the holder becomes incapacitated or passes away.

We live in a deeply unstable world. There are a lot of ongoing conflicts around the world have made this reality
impossible to ignore: soldiers and civilians alike can lose their lives in a random moment — to a missile strike, a
landmine, or a frontline engagement. But the urgency extends far beyond any single conflict.

The question is urgent and personal: **How do I ensure that a designated loved ones like my spouse, my children, my
family can eventually access what I leave behind? Without exposing them prematurely, without trusting the intermediary
server, and without requiring complex multi-party coordination...**

Existing solutions fail this need. Trusting a centralized custodian means surrendering control to a third party.
Physical storage — paper in a safe, a USB drive in a deposit box — can be destroyed, seized, or simply inaccessible when
it matters most.

BitHeritage solves this with a solution that combines:

- **End-to-end encryption** using the recipient's public key, so the server never sees plain data.
- **Double-layer encryption** where the server adds its own encryption layer, preventing data exfiltration even if the
  database is compromised.
- **A grace period mechanism** that gives the secret holder time to deny any unauthorized access attempt.
- **Passwordless authentication** via WebAuthn passkeys, eliminating phishing and credential stuffing attacks.
- **Maximum Security** your data never leaves your device and protected by your biometry.

---

## Design Principles

### Zero Knowledge Identity

The platform never collects personally identifiable information. Users register with a self-chosen tag name (e.g.,
`@alice`) and a device-bound passkey. The server stores only the tag name and the WebAuthn public key credentials. There
are no emails, phone numbers, or passwords.

### End-to-End Double Encryption

Secrets are encrypted on the sender's device using the recipient's public key before transmission and never leave the
device. The server cannot decrypt the inner layer.

A second encryption layer is applied server-side. Even if an attacker exfiltrates the database, they obtain only
doubly encrypted ciphertext that is useless without both the server's master key and the recipient's private key.

### Defense in Depth

The BitHeritage app never stores encryption key material on the device. Instead, it uses the WebAuthn
[PRF extension](https://w3c.github.io/webauthn/#prf-extension) to deterministically derive encryption keys from the
authenticator's internal secret on every session. The derived key exists only in memory for the duration of the
cryptographic operation and is never written to disk, local storage, or any persistent medium. Access to the PRF output
requires biometric or PIN verification through the authenticator, ensuring that encryption keys are protected by the
same hardware-backed security as authentication.

### Owner Sovereignty

The secret owner (Alice) retains full control at all times. They can revoke access and delete shared secrets, or deny
inheritance requests — up until the grace period expires without intervention. Grace period is 6 months.

### Don't Trust – VERIFY

The entire system — client applications and server services — is open-source and can be audited by everyone.

### Verifiable Deployments

Everything is deployed with cryptographic attestations to make sure that the running code matches the published source,
eliminating the need to trust the operator's claims.

### Minimal Attack Surface

Passwordless authentication via passkeys eliminates the most common attack vectors: phishing, credential stuffing,
password reuse, and database breaches that expose hashed passwords. The device-bound nature of passkeys means
authentication cannot be replayed from another device.

### Smart Secure Server Side Architecture

The server-side storage service that contains doubly-encrypted data and the stateless encryption service are
physically isolated. The encryption service retrieves its master key from an external **Secrets Manager** at startup and
holds it exclusively in locked memory.
A unique per-secret encryption key is derived from the master key for each secret, so that no single key protects more
than one secret.

---

_See [Architecture](ARCHITECTURE.md) for more details._

---

## User Flow

### Registration

1. User opens the BitHeritage app and chooses a tag name (e.g., `@alice`).
2. The app initiates a WebAuthn registration ceremony with the device's authenticator (Touch ID, Face ID, Windows Hello,
   etc.) and requests the PRF extension.
3. The authenticator generates a device-bound passkey (ECDSA P-256 keypair).
4. The app evaluates the PRF extension with a fixed domain-separation salt to derive 32 bytes of key material.
5. The app derives an X25519 encryption keypair from the PRF output using HKDF-SHA256. The private key is held only in
   memory and never stored.
6. The WebAuthn public key credential and the X25519 encryption public key are sent to the server.
7. The server stores the tag name, WebAuthn credential, and encryption public key.

**Zero-knowledge guarantee:** The server learns only the tag name, an authentication public key, and an encryption
public key. No private key material ever leaves the device!

### Authentication

1. User enters their tag name.
2. The server issues a WebAuthn authentication challenge.
3. The user's device signs the challenge with the stored passkey.
4. The server verifies the signature against the stored public key.
5. A session token is issued.

No passwords, no OTPs, no recovery emails.

### Sharing a Secret

Alice wants to share a secret with Bob:

1. Alice selects Bob's tag name as the counterparty.
2. The app fetches Bob's **encryption public key** (X25519) from the server.
3. **Client-side encryption (Layer 1):**
    - The app generates an ephemeral X25519 keypair.
    - Performs X25519 key agreement between the ephemeral private key and Bob's encryption public key to produce a
      shared
      secret.
    - Derives a symmetric encryption key using HKDF-SHA256 with the shared secret as input, the concatenation of the
      ephemeral public key and Bob's public key as salt, and a fixed context string as info.
    - Encrypts the plaintext with XChaCha20-Poly1305 using the derived key and random 24-byte nonce.
4. The encrypted payload (ciphertext + ephemeral public key + nonce) is sent to the server.
5. **Server-side encryption (Layer 2):** The encryption service derives a unique per-secret key from the master key (
   using HKDF-SHA256 with the secret's UUID as salt) and wraps the already-encrypted payload with ChaCha20-Poly1305.
6. The doubly encrypted secret is stored in the database.

### Requesting a Secret (Inheritance Claim)

1. Bob initiates an inheritance request for a secret shared by Alice.
2. The server records the request and starts the **grace period timer** (default: 6 months).
3. Alice receives push notifications alerting her that Bob has requested access.
4. During the grace period:
    - Alice can **deny** the request at any time, canceling the transfer.
    - Alice can **delete** the shared secret entirely.
    - Bob waits.
5. If the grace period expires without denial from Alice:
    - The server decrypts Layer 2 encryption (its own layer).
    - The Layer 1 ciphertext is transmitted to Bob.
    - Bob authenticates with his passkey and the app evaluates the PRF extension to re-derive his X25519 private key.
    - Bob's app performs X25519 key agreement between his private key and the ephemeral public key to reconstruct the
      shared secret, derives the same symmetric key via HKDF-SHA256, and decrypts the ciphertext.
    - Bob reads the plaintext secret.

---

## Grace Period and Access Control

### Grace Period Mechanism

The grace period is the core safety mechanism that distinguishes BitHeritage from simple encrypted storage.
It creates a mandatory waiting window during which the secret owner can intervene.

**Default duration:** 6 months (180 days).

**Rationale:** The grace period must be long enough that an incapacitated owner's absence is noticeable and matches
common probate timelines.

### Notification Mechanism

When Bob requests access:

1. **Immediate:** Push notification to Alice's registered device(s).
2. **Periodic:** Recurring notifications at configurable intervals throughout the grace period.
3. **Escalation:** Increased notification frequency in the final 30 days.

If Alice's device is unreachable (device lost, destroyed), the notifications serve as a dead-man's switch:
the absence of denial is treated as implicit consent after the full grace period.

### Owner Actions During Grace Period

- **Deny request:** Immediately cancels the transfer. Bob is notified. A new request restarts the full grace period.
- **Delete secret:** Permanently removes the encrypted secret from the server. The inheritance request is void.
- **No action:** After 180 days, the server proceeds with transfer.

### Device Recovery

If the user loses their device, re-registration with the same tag name is not possible.
The Webauthn authenticator device recovery is out of scope for this project,
but practical recommendations will be provided in the future.

---

## Trust Model and Threat Analysis

### What the Server Knows

- Tag names, WebAuthn public key credentials, and X25519 encryption public keys.
- Doubly encrypted secret ciphertext.
- Which users have shared secrets with whom (relationship metadata).
- Timing of requests and transfers.

### What the Server Cannot Do

- Decrypt Layer 1 ciphertext (requires the recipient's X25519 private key) and access any secret's plaintext.
- Impersonate users.
- Transfer a secret before the grace period expires (enforced by application logic).

### Threat Scenarios

- **Database breach** — Double encryption renders exfiltrated data useless. Each secret's Layer 2 is encrypted with a
  unique per-secret key derived from the master key, so even if the database is fully exfiltrated, the attacker needs
  both the master key (held only in the Secrets Manager and Encryption Service memory) and each recipient's private key.
- **Server compromise (full)** — Attacker gains the master key and can remove Layer 2, but Layer 1 still requires each
  recipient's X25519 private key, which is derived from the authenticator's PRF secret and never stored on the server.
  No single breach exposes all secrets.
- **Malicious server operator** — Verifiable deployment allows clients to confirm server code matches the open-source
  repository. A malicious operator would need to deploy a modified code, which attestation would detect.
- **Stolen recipient device** — Attacker must also wait for a grace period or compromise Alice's device to prevent
  denial.
- **Social engineering (fake claim)** — Grace period + push notifications give Alice 6 months to detect and deny
  fraudulent requests.
- **Compromised sender device** — If Alice's device is compromised before sharing, the attacker could read secret before
  encryption anyway. The device must be trusted at the moment of encryption.

### Limitations

- **Metadata visibility:** The server knows who shared secrets with whom. Future versions may explore metadata-hiding
  techniques.
- **Single-server dependency:** The current design relies on a single server operator. Federation or decentralized
  storage may be explored in future versions.
- **Device loss without recovery:** If both Alice and Bob lose their devices and cannot complete recovery, encrypted
  secrets are permanently inaccessible by design.

---

## Roadmap

### Phase 1: Foundation (Current — PoC)

Dates: Q1–Q2 2026

- [x] Microservice architecture (gateway, encryption, storage)
- [x] WebAuthn passkey registration and authentication
- [x] Secret encryption and storage
- [x] Basic inheritance request and grace period model
- [ ] Grace period scheduler (automated enforcement)
- [ ] Service-to-service authentication (mTLS)

### Phase 2: Client Applications

Dates: Q2 2026

- [ ] React web application with full WebAuthn integration
- [ ] React Native mobile applications (iOS, Android)
- [ ] Client-side encryption
- [ ] Push notification delivery

### Phase 3: Verifiable Deployment

Dates: Q3 2026

- [ ] GitHub Actions CI/CD pipeline with artifact attestations
- [ ] Server `/attestation` endpoint
- [ ] Client-side attestation verification
- [ ] Reproducible builds

### Phase 4: Hardening

Dates: Q3–Q4 2026

- [ ] Comprehensive test suite (unit, integration, end-to-end)
- [ ] Security audit by independent third party
- [ ] Rate limiting, abuse prevention
- [ ] Key rotation mechanism for the server master key (via Secrets Manager versioning)
- [ ] Device recovery protocol

### Phase 5: Open Launch

Dates: Q4 2026

- [ ] Open-source release under permissive license
- [ ] Public documentation and API reference
- [ ] Community feedback and protocol evolution

---

## Conclusion

BitHeritage provides a practical solution to the digital inheritance problem. By combining WebAuthn passwordless
authentication, PRF-derived X25519 end-to-end encryption, server-side double encryption, and a grace-period safety
mechanism, the protocol ensures that secrets are accessible only to the intended recipient and only after the owner has
had ample opportunity to intervene.

The open-source, verifiably deployed nature of the system means that trust is grounded in mathematics and transparency
rather than promises.

Digital secrets are too important to be lost. BitHeritage ensures they outlive their holders — securely, privately, and
on the holder's terms.

---

*BitHeritage is open-source software. Contributions, security reviews, and feedback are welcome.*

# BitHeritage — Architecture

**Version 0.1 — Initial Draft**

---

## Table of Contents

1. [System Context](#system-context)
2. [Container Architecture](#container-architecture)
3. [Component Architecture](#component-architecture)
4. [Network Topology](#network-topology)
5. [Cryptographic Architecture](#cryptographic-architecture)
6. [Data Model](#data-model)
7. [Key Flows](#key-flows)
8. [Verifiable Deployment](#verifiable-deployment)
9. [Technology Stack](#technology-stack)

---

## System Context

The C4 Level 1 (Context) diagram shows BitHeritage and the external actors that interact with it.

[![arch-1-c4-context.puml](https://tinyurl.com/27q2tluw)](https://tinyurl.com/27q2tluw)<!--![arch-1-c4-context.puml](puml/arch-1-c4-context.puml)-->

**Actors:**

- **Secret Owner (Alice)** — Registers with a passkey, encrypts secrets on-device, and shares them with a designated
  inheritor. Retains full control (deny/delete) until the grace period expires.
- **Inheritor (Bob)** — Registers with a passkey, requests access to shared secrets, and decrypts them on-device after
  the grace period.

**External Systems:**

- **Push Notification Service** — Apple Push Notification Service (APNs) and Firebase Cloud Messaging (FCM) deliver
  alerts to Alice when Bob initiates an inheritance claim.
- **Secrets Manager** — An external key management service (HashiCorp Vault, AWS Secrets Manager, or GCP Secret
  Manager) that securely stores and provisions the server-side master encryption key to the Encryption Service.

---

## Container Architecture

The C4 Level 2 (Container) diagram shows the major deployable units.

[![arch-2-c4-container.puml](https://tinyurl.com/2yet5g6c)](https://tinyurl.com/2yet5g6c)<!--![arch-2-c4-container.puml](puml/arch-2-c4-container.puml)-->


### Client Side

| Component                   | Description                                                                                                                                                                                                               |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Compatible Device**       | Any device supporting the WebAuthn standard — iPhone (Touch ID / Face ID), Android (fingerprint / face unlock), desktop browser (Windows Hello, security key). The device's secure enclave stores passkey material.       |
| **WebAuthn Authenticator**  | The platform authenticator built into the device OS.                                                                                                                                                                      |
| **BitHeritage Application** | The user-facing app (mobile or web). Orchestrates registration, authentication, secret sharing, and inheritance requests. Derives encryption keys from the authenticator's PRF extension and performs all Layer 1 encryption/decryption on-device. |

### Server Side

All server-side services run within a **logically isolated network**. Only the Gateway Service exposes public endpoints.
The Encryption Service and Storage Service are unreachable from the outside.

| Component              | Description                                                                                                                                                                                               |
|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Gateway Service**    | Exposes REST API endpoints over HTTPS. Handles WebAuthn registration/authentication ceremonies, session management, request routing to internal services.                                                 |
| **Encryption Service** | Internal-only, stateless service. Retrieves the master key from the Secrets Manager at startup, derives per-secret keys via HKDF, and applies or removes Layer 2 encryption. The master key is held exclusively in locked memory. |
| **Storage Service**    | Internal-only, stateful service. Provides a CRUD interface over PostgreSQL. Stores users, doubly-encrypted secrets, inheritance transfer records, and notifications. Enforces data integrity constraints. |
| **Secrets Manager**    | External service (HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager). Stores the server master key with versioning, access control, and audit logging. Only the Encryption Service is authorized to read the key. |
| **PostgreSQL**         | The persistent data store.                                                                                                                                                                                |

---

[//]: # (TODO add component architecture diagram)

## Cryptographic Architecture

### Key Types

| Key | Holder | Purpose |
|---|---|---|
| **User Passkey (ECDSA P-256)** | Device-bound (secure enclave) | WebAuthn authentication only |
| **PRF Secret (32 bytes)** | Derived from authenticator via PRF extension, held in memory only | Seed material for encryption keypair derivation |
| **Encryption Keypair (X25519)** | Private: re-derived from PRF each session, never stored. Public: stored on server | ECDH key agreement for Layer 1 encryption |
| **Ephemeral Keypair (X25519)** | Generated per secret, discarded after use | Key agreement for Layer 1 encryption |
| **Layer 1 Symmetric Key (256-bit)** | Computed on device via HKDF-SHA256 | XChaCha20-Poly1305 encryption for Layer 1 |
| **Server Master Key (256-bit)** | Encryption service (loaded from Secrets Manager at startup, held in memory only) | Root key for deriving per-secret Layer 2 encryption keys |
| **Per-Secret Key (256-bit)** | Derived in memory per encryption/decryption operation, never stored | ChaCha20-Poly1305 encryption for Layer 2 |

### Key Separation Principle

The WebAuthn passkey (ECDSA P-256) is used **exclusively for authentication** — signing challenges during login. It is never used for encryption or key agreement. 
A separate X25519 encryption keypair handles all cryptographic operations related to secret sharing. 
This separation ensures that:

- The authentication key remains in the secure enclave and is never exposed to application code.
- The encryption key is derived deterministically from the authenticator's
  [PRF extension](https://w3c.github.io/webauthn/#prf-extension) output, which itself requires biometric/PIN
  verification to access.

### PRF-Based Encryption Key Derivation

During registration and each subsequent authentication, the app evaluates the WebAuthn PRF extension to derive the user's encryption keypair:

1. The app provides a fixed domain-separation salt to the PRF extension:
   `eval: { first: SHA-256("bitheritage:encryption:v1") }`.
2. The authenticator's internal HMAC-secret function produces a deterministic 32-byte PRF output, bound to this
   specific credential.
3. The app imports the 32-byte PRF output as input keying material (IKM) and derives an X25519 private key using
   HKDF-SHA256: `HKDF-SHA256(ikm=PRF_output, salt=empty, info="bitheritage-x25519-key-v1", len=32)`.
4. The corresponding X25519 public key is computed from the private key.
5. During registration, the public key is sent to the server for storage.
6. The private key is held in memory only for the duration of the session and is never written to disk, local storage, or any persistent medium.

_Because the PRF output is deterministic (same credential + same salt = same output), the same X25519 keypair is
re-derived on every session without requiring any stored key material._

### Layer 1: Client-Side Encryption (X25519 + XChaCha20-Poly1305)

For each secret, the sender's device:

1. Fetches the recipient's X25519 encryption public key from the server.
2. Generates an ephemeral X25519 keypair.
3. Performs X25519 key agreement between the ephemeral private key and the recipient's public key to produce a shared secret.
4. Derives a symmetric encryption key:
   `HKDF-SHA256(ikm=shared_secret, salt=ephemeral_public || recipient_public, info="bitheritage-layer1-v1", len=32)`.
5. Encrypts the plaintext with XChaCha20-Poly1305 using the derived key and random 24-byte nonce.
6. Discards the ephemeral private key.
7. Packages the ciphertext, ephemeral public key, and nonce for transmission to the server.

_Only the recipient, possessing the X25519 private key (re-derived from their authenticator's PRF secret), can perform the reverse key agreement to derive the same shared secret and decrypt the data._

### Layer 2: Server-Side Encryption

The Encryption Service derives a unique per-secret key for each encryption operation, ensuring that a single key never protects more than one secret:

1. Gateway Service receives the Layer 1 ciphertext and the secret's UUID from the client and passes both to the
   Encryption Service.
2. Encryption Service derives a per-secret key:
   `HKDF-SHA256(ikm=master_key, salt=secret_id, info="bitheritage-layer2-v1", len=32)`.
3. Encryption Service encrypts the Layer 1 ciphertext with ChaCha20-Poly1305 using the per-secret key and a random nonce.
4. Gateway Service sends the re-encrypted ciphertext and the Layer 2 nonce for storage.

During decryption (inheritance transfer), the same derivation is performed in reverse: the secret's UUID is used to
re-derive the same per-secret key and decrypt the Layer 2 envelope.

### Master Key Management

The server master key is the root of trust for all Layer 2 encryption. It is managed through an external **Secrets Manager** (e.g., HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager):

- **Provisioning:** At startup, the Encryption Service authenticates to the Secrets Manager and retrieves the master
  key. The key is held exclusively in locked memory (`mlock`) and is never written to disk, logs, environment
  variables, or any persistent medium.
- **Isolation:** The Secrets Manager is the only component that persistently stores the master key. The Encryption
  Service, Storage Service, Gateway Service, and database never have access to it.
- **Key rotation:** The Secrets Manager supports versioned secrets. During rotation, the Encryption Service loads both
  the current and previous master key versions. New secrets are encrypted with the current version; existing secrets
  are decrypted using the version recorded in their metadata and re-encrypted with the current version on access.
- **Access control:** Only the Encryption Service's identity is authorized to read the master key from the Secrets Manager. All access is audited.

### Double Encryption Flow

[![arch-3-encryption-flow.puml](https://tinyurl.com/23akbxy6)](https://tinyurl.com/23akbxy6)<!--![arch-3-encryption-flow.puml](puml/arch-3-encryption-flow.puml)-->

---

## Data Model

[![arch-4-data-model.puml](https://tinyurl.com/2dy99cbs)](https://tinyurl.com/2dy99cbs)<!--![arch-4-data-model.puml](puml/arch-4-data-model.puml)-->

**Constraints:**

- `secrets` has a unique constraint on `(owner_id, inheritor_id, title)` — one secret per owner-inheritor-title
  combination.
- `users.encryption_public_key` stores the X25519 public key derived from the authenticator's PRF extension during registration. This key is used by counterparties to encrypt secrets for this user.
- `secrets.ephemeral_public_key` and `secrets.nonce` store the per-secret cryptographic parameters required for Layer 1 decryption.
- `secrets.l2_nonce` and `secrets.l2_key_version` store the Layer 2 encryption nonce and the master key version used,
  enabling key rotation without re-encrypting all existing secrets at once.
- Deleting a user cascades to their secrets, transfers, and notifications.
- Deleting a secret cascades to its transfers and related notifications.

---


## Verifiable Deployment

Trust in BitHeritage should not rely on the operator's reputation. The project adopts a **verifiable deployment** model.

### Open Source

All source code — client apps and server services — is published on GitHub under an open-source license. Anyone can
audit the cryptographic implementations, data handling, and access control logic.

### Build Attestations

Releases are built via GitHub Actions
with [artifact attestations](https://docs.github.com/en/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds).
Each build produces a signed provenance record that links the binary artifact to a specific source commit, build
environment, and workflow definition.

### Deployment Verification

[//]: # (TODO add Deployment Verification)

---

## Technology Stack

| Component             | Technology                                 | Rationale                                                                                                                        |
|-----------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| **Server language**   | Rust                                       | Memory safety, performance, strong type system. Eliminates entire classes of vulnerabilities (buffer overflows, use-after-free). |
| **Web framework**     | Axum                                       | Async, tower-based middleware ecosystem, first-class Rust support.                                                               |
| **Database**          | PostgreSQL                                 | Battle-tested relational database with strong data integrity guarantees, JSONB support for WebAuthn credentials.                 |
| **Cryptography**      | `ring` (Rust) / WebCrypto + WebAuthn PRF (client) | `ring` is a well-audited, high-performance crypto library. Client uses WebAuthn PRF for key derivation, WebCrypto (HKDF, X25519, XChaCha20-Poly1305) for encryption. |
| **Client (mobile)**   | React Native or Dart (Flutter)             | Cross-platform mobile development for iOS and Android from a single codebase.                                                    |
| **Client (web)**      | React                                      | Browser-based interface using the Web Authentication API.                                                                        |
| **CI/CD**             | GitHub Actions                             | Native integration with GitHub, artifact attestation support.                                                                    |
| **Secrets management** | HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager | Centralized master key storage with versioning, access control, and audit logging. Eliminates environment variable key provisioning. |
| **Container runtime** | Docker                                     | Reproducible builds, isolated deployment.                                                                                        |

---

## Open Questions and Improvement Proposals

### Consider Distributed Architecture based on Secret Sharing

Consider implementing a distributed architecture that leverages secret sharing techniques, 
for example, [SSS](https://en.wikipedia.org/wiki/Shamir%27s_secret_sharing) to enhance security and resilience.

### Consider Encryption Improvements

Context: now Alice should trust the server and can't verify that the Bob's public key is really belongs to Bob.
I.e. the server acts as a certificate authority.

To solve this problem, consider implementing [Chaum-Pedersen protocol/](https://muens.io/chaum-pedersen-protocol/) go guarantee the encryption consistency.


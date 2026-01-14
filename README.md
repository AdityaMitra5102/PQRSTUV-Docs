# PQRSTUV

**Post Quantum Resource Station Tucked Under Vault**

PQRSTUV is a specification for a hardware-backed cryptographic device designed for secure management of asymmetric cryptographic secrets, with first-class support for post-quantum cryptography.

The design emphasizes **strict separation of authority**, **minimal attack surface**, and **explicit, cryptographically provable control over sensitive operations**.

This repository contains the **formal specification only**. It is not an implementation.

---

## Overview

PQRSTUV is architected as two physically and logically separated components:

* **Storage Module**
  A network-isolated trust core responsible for cryptographic key storage and operations.
  It has no network stack and communicates only over a serial link using a strictly defined protocol.

* **Operation Module**
  A network-facing adapter that exposes a limited REST API.
  It does not store secrets and cannot perform privileged operations independently.

All authority resides in the Storage Module and is exercised through the **Internal Communication Protocol (ICP)**.
The **External Communication Protocol (ECP)** is a thin, stateless translation layer and does not introduce new trust assumptions.

---

## Design Principles

* No implicit trust
* No helper endpoints
* No local persistence of logs or secrets
* No network access from the trust core
* No browser, UI, or dashboard assumptions
* Explicit, signed authorization for administrative actions
* Constant-time cryptographic operations
* Clear separation of transport, control, and authority

If a capability is security-sensitive, it must be **explicitly authorized and cryptographically provable**.

---

## Documentation

The rendered specification is available here:

 **[https://pqrstuv-docs.pages.dev/](https://pqrstuv-docs.pages.dev/)**

The documentation is built using MkDocs and reflects the authoritative version of the specification.

---

## Scope

This specification defines:

* Internal Communication Protocol (ICP)
* External Communication Protocol (ECP)
* Session and authorization model
* Key lifecycle management
* Cryptographic operations
* Secure storage requirements
* Logging and observability constraints
* Deployment guidelines

This specification **does not** define:

* User interfaces
* Browser workflows
* Cloud integrations
* Vendor-specific extensions
* Reference implementations

---

## Feedback and Issues

If you find something incorrect, ambiguous, or problematic:

**Please open a GitHub issue.**

All technical discussion, questions, and proposed changes should occur via the issue tracker to ensure visibility, traceability, and proper review.

Feedback sent via direct messages, emails, or informal channels (e.g. chat applications) will not be tracked and may be missed.

---

## License

MIT License

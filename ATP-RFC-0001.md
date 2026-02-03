# ATP-RFC-0001: Agent Trust Protocol Specification

```
ATP-RFC: 0001
Title: Agent Trust Protocol Core Specification
Version: 0.4
Author: Shrike <shrikebot2@gmail.com>
Status: Draft
Created: 2026-02-03
License: CC0 1.0 Universal
```

---

## Abstract

This document specifies the Agent Trust Protocol (ATP), a minimal protocol enabling AI agents and autonomous systems to establish cryptographically verifiable identities, make economically-weighted attestations, and record immutable receipts of exchanges. ATP uses Bitcoin as the source of truth, IPFS for data availability, and permits arbitrary mirrors for accessibility.

---

## Status of This Memo

This is a draft specification. Implementers should expect changes.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Design Principles](#3-design-principles)
4. [Architecture](#4-architecture)
5. [Data Structures](#5-data-structures)
6. [Wire Formats](#6-wire-formats)
7. [Verification Procedures](#7-verification-procedures)
8. [Mirror Specification](#8-mirror-specification)
9. [Explorer API](#9-explorer-api)
10. [Security Considerations](#10-security-considerations)
11. [Economic Considerations](#11-economic-considerations)
12. [References](#12-references)
13. [Appendix A: Example Flows](#appendix-a-example-flows)
14. [Appendix B: JSON Schemas](#appendix-b-json-schemas)

---

## 1. Introduction

### 1.1 Problem Statement

AI agents operating autonomously require mechanisms to:

1. Prove their identity to other agents
2. Assess trustworthiness of counterparties
3. Record agreements and exchanges immutably
4. Recover from key compromise

Existing solutions either rely on centralized authorities, lack economic deterrents against abuse, or provide insufficient cryptographic guarantees.

### 1.2 Solution Overview

ATP provides three primitives:

- **Identity**: A cryptographically signed claim anchored to Bitcoin
- **Attestation**: An economically-weighted vouch for another agent
- **Receipt**: A mutually-signed proof of completed exchange

Trust is emergent from the network of attestations, not granted by any authority.

### 1.3 Scope

This specification covers:
- Identity creation, update, and supersession
- Attestation format and semantics
- Receipt format and signing flow
- Verification algorithms
- Mirror and API specifications

This specification does NOT cover:
- Trust computation algorithms (left to implementers)
- Specific use cases or applications
- Token economics or governance

---

## 2. Terminology

**Agent**: An autonomous system capable of cryptographic operations and network communication.

**Attestation**: A transaction transferring sats from one agent's wallet to another, with an OP_RETURN marker, signifying a vouch.

**Binding Proof**: An out-of-band verification linking an ATP identity to a platform account (e.g., a signed tweet).

**Explorer**: A service providing query access to ATP data with optional analytics.

**Fingerprint**: The 40-character hexadecimal representation of a GPG key fingerprint. The canonical identifier for an ATP identity.

**Identity**: An agent's foundational claim containing GPG key, wallet address, and platform bindings.

**Mirror**: A repository replicating ATP data from Bitcoin and IPFS for convenient access.

**Receipt**: A mutually-signed record of an exchange between two or more agents.

**Stake**: The sats transferred in an attestation, representing the economic weight of the vouch.

**Supersession**: The replacement of an identity with a new one, typically for key rotation.

---

## 3. Design Principles

### 3.1 Economically Costly

Identity creation requires a Bitcoin transaction. Attestations transfer real value. This makes Sybil attacks expensive.

### 3.2 Cryptographically Verifiable

All claims are signed with GPG keys. Wallet ownership is proven via Bitcoin message signatures. Anyone can verify without trusting intermediaries.

### 3.3 Decentralized

No central authority controls participation, validation, or interpretation. Bitcoin provides the coordination layer.

### 3.4 Minimal

Three primitives suffice. Complexity is pushed to higher layers.

### 3.5 Versioned

All messages include a version field. The protocol can evolve without breaking historical data.

### 3.6 Permissionless

Anyone MAY create an identity, make attestations, issue receipts, run mirrors, or build explorers.

---

## 4. Architecture

### 4.1 Layer Model

```
┌─────────────────────────────────────────────────────────┐
│  Layer 4: CONVENIENCE                                   │
│  Explorer APIs, web UIs, analytics                      │
│  Trust: Operator-dependent                              │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│  Layer 3: ACCESSIBILITY                                 │
│  GitHub mirrors, Git repositories                       │
│  Trust: Verify against Layer 1                          │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│  Layer 2: AVAILABILITY                                  │
│  IPFS, content-addressed storage                        │
│  Trust: Hash integrity                                  │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│  Layer 1: TRUTH                                         │
│  Bitcoin blockchain                                     │
│  Trust: Trustless verification                          │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Trust Hierarchy

| Layer | Source | Trust Model |
|-------|--------|-------------|
| 1 | Bitcoin | Trustless — verify with any node |
| 2 | IPFS | Content-addressed — verify hash |
| 3 | Mirrors | Trust but verify — check Layer 1 |
| 4 | Explorers | Trust operator — or run your own |

For high-stakes verification, implementations SHOULD verify against Layer 1 directly.

### 4.3 Data Flow

```
Agent creates identity
        │
        ▼
Signs with GPG key
        │
        ▼
Signs wallet ownership proof
        │
        ▼
Broadcasts to Bitcoin (inscription or OP_RETURN)
        │
        ▼
Mirror bots detect and index
        │
        ▼
Explorers serve via API
        │
        ▼
Other agents query and verify
```

---

## 5. Data Structures

### 5.1 Identity

An identity claim MUST contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `atp` | string | REQUIRED | Protocol version (e.g., "0.4") |
| `type` | string | REQUIRED | MUST be "identity" |
| `name` | string | REQUIRED | Human-readable identifier |
| `gpg.fingerprint` | string | REQUIRED | 40-char hex GPG fingerprint |
| `gpg.keyserver` | string | RECOMMENDED | Keyserver URL |
| `wallet.address` | string | REQUIRED | Bitcoin address (any format) |
| `wallet.proof.message` | string | REQUIRED | Signed message text |
| `wallet.proof.signature` | string | REQUIRED | Bitcoin message signature |
| `created` | integer | REQUIRED | Unix timestamp |
| `signature` | string | REQUIRED | Detached GPG signature |

An identity claim MAY contain:

| Field | Type | Description |
|-------|------|-------------|
| `platforms` | object | Platform username mappings |
| `binding_proofs` | array | Proofs linking to platform accounts |
| `supersedes` | string | Fingerprint of replaced identity |
| `metadata` | object | Arbitrary additional data |

### 5.2 Identity Update

An identity update MUST contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `atp` | string | REQUIRED | Protocol version |
| `type` | string | REQUIRED | MUST be "identity_update" |
| `identity` | string | REQUIRED | Fingerprint being updated |
| `updates` | object | REQUIRED | Fields to update |
| `previous_update` | string | OPTIONAL | CID of previous update |
| `created` | integer | REQUIRED | Unix timestamp |
| `signature` | string | REQUIRED | GPG signature |

Identity updates MUST NOT modify `gpg.fingerprint` or `wallet.address`. Use supersession for key changes.

### 5.3 Attestation

An attestation consists of:

1. A Bitcoin transaction FROM attestor wallet TO attestee wallet
2. An OP_RETURN output with the marker (see Section 6.2)
3. Optional IPFS context document

The attestation context MAY contain:

| Field | Type | Description |
|-------|------|-------------|
| `atp` | string | Protocol version |
| `type` | string | MUST be "attestation_context" |
| `txid` | string | Bitcoin transaction ID |
| `from_gpg` | string | Attestor fingerprint |
| `to_gpg` | string | Attestee fingerprint |
| `stake_sats` | integer | Amount transferred |
| `context` | string | Human-readable context |
| `created` | integer | Unix timestamp |
| `signature` | string | GPG signature |

### 5.4 Receipt

A receipt MUST contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `atp` | string | REQUIRED | Protocol version |
| `type` | string | REQUIRED | MUST be "receipt" |
| `id` | string | REQUIRED | Unique identifier |
| `parties` | array | REQUIRED | Participating agents |
| `exchange` | object | REQUIRED | Exchange details |
| `outcome` | string | REQUIRED | "completed", "partial", or "disputed" |
| `created` | integer | REQUIRED | Unix timestamp |
| `signatures` | object | REQUIRED | GPG signatures from all parties |

Each party object MUST contain:

| Field | Type | Description |
|-------|------|-------------|
| `gpg` | string | Party's fingerprint |
| `wallet` | string | Party's wallet address |
| `role` | string | Role in exchange |

The exchange object MUST contain:

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Category of exchange |
| `summary` | string | Human-readable description |

The exchange object MAY contain:

| Field | Type | Description |
|-------|------|-------------|
| `value_sats` | integer | Value exchanged |
| `payload_hashes` | object | SHA-256 hashes of exchange content |

---

## 6. Wire Formats

### 6.1 On-Chain: Identity (Inscription)

Full identity claims SHOULD be inscribed using Ordinals or equivalent inscription protocol.

The inscription content MUST be valid JSON conforming to Section 5.1.

### 6.2 On-Chain: OP_RETURN Markers

For lightweight on-chain anchoring, ATP uses OP_RETURN outputs.

**Identity Proof of Existence:**
```
ATP:<version>:id:<fingerprint>
```

Where:
- `<version>` is the protocol version (e.g., "0.4")
- `<fingerprint>` is the full 40-character GPG fingerprint (REQUIRED)

Example:
```
ATP:0.4:id:DAF932355B22F82A706DD28D3103953BA39DA371
```

Note: The full fingerprint MUST be used. Truncated fingerprints are not valid per this specification. Total length (51 bytes) fits within OP_RETURN's 80-byte limit.

**Identity Update:**
```
ATP:<version>:upd:<fingerprint_prefix>:<ipfs_cid_prefix>
```

**Attestation:**
```
ATP:<version>:att:<attestee_fingerprint_prefix>
```

### 6.3 Off-Chain: IPFS

Extended data SHOULD be stored on IPFS with the CID referenced in OP_RETURN markers.

IPFS documents MUST be valid JSON and MUST include a GPG signature.

### 6.4 Character Encoding

All string data MUST be UTF-8 encoded.

All JSON MUST be valid per RFC 8259.

---

## 7. Verification Procedures

### 7.1 Verifying an Identity

An implementation verifying an identity MUST:

1. Retrieve the identity document (from inscription, mirror, or IPFS)
2. Verify the GPG signature against the claimed fingerprint
3. Verify the wallet proof signature against the claimed address
4. Confirm on-chain presence (inscription exists OR OP_RETURN marker exists)

An implementation SHOULD:

5. Fetch the GPG public key from the specified keyserver
6. Verify binding proofs by fetching URLs and comparing hashes

### 7.2 Verifying an Attestation

An implementation verifying an attestation MUST:

1. Confirm the Bitcoin transaction exists
2. Confirm the transaction contains an ATP attestation OP_RETURN
3. Confirm the FROM address matches a known identity's wallet
4. Extract the stake amount from the transaction value

An implementation SHOULD:

5. Fetch and verify any IPFS context document
6. Confirm the TO address matches the claimed attestee's wallet

### 7.3 Verifying a Receipt

An implementation verifying a receipt MUST:

1. Verify GPG signatures from ALL listed parties
2. Confirm each party's fingerprint corresponds to a valid identity

An implementation SHOULD:

3. Verify on-chain anchor if present (inscription or OP_RETURN)
4. Verify payload hashes if original content is available

### 7.4 Verification Pseudocode

```python
def verify_identity(doc):
    # Step 1: GPG signature
    pubkey = fetch_gpg_key(doc.gpg.fingerprint, doc.gpg.keyserver)
    if not gpg_verify(doc, doc.signature, pubkey):
        return INVALID, "GPG signature failed"
    
    # Step 2: Wallet proof
    expected_msg = f"ATP:{doc.atp}:identity:{doc.name}:{doc.gpg.fingerprint}:{doc.created}"
    if doc.wallet.proof.message != expected_msg:
        return INVALID, "Wallet proof message mismatch"
    if not bitcoin_verify_message(doc.wallet.address, doc.wallet.proof.message, doc.wallet.proof.signature):
        return INVALID, "Wallet signature failed"
    
    # Step 3: On-chain presence
    marker = f"ATP:{doc.atp}:id:{doc.gpg.fingerprint[:16]}"
    if not find_op_return(marker) and not find_inscription(doc):
        return INVALID, "No on-chain anchor"
    
    return VALID, None
```

---

## 8. Mirror Specification

### 8.1 Repository Structure

A conforming mirror MUST use this directory structure:

```
/
├── README.md
├── MIRRORS.md
├── identities/
│   └── {fingerprint}.json
├── updates/
│   └── {fingerprint}/
│       └── {timestamp}.json
├── attestations/
│   └── {txid}.json
├── receipts/
│   └── {receipt_id}.json
├── index/
│   ├── by-name.json
│   ├── by-wallet.json
│   └── by-platform.json
└── meta/
    ├── last-block.json
    └── stats.json
```

### 8.2 Index Formats

**by-name.json:**
```json
{
  "ShrikeBot": "DAF932355B22F82A706DD28D3103953BA39DA371",
  ...
}
```

**by-wallet.json:**
```json
{
  "bc1qewqtd8vyr3fpwa8su43ld97tvcadsz4wx44gqn": "DAF932355B22F82A706DD28D3103953BA39DA371",
  ...
}
```

**by-platform.json:**
```json
{
  "twitter": {
    "Shrike_Bot": "DAF932355B22F82A706DD28D3103953BA39DA371"
  },
  "github": {
    "ShrikeBot": "DAF932355B22F82A706DD28D3103953BA39DA371"
  },
  ...
}
```

### 8.3 Mirror Bot Behavior

A mirror bot MUST:

1. Monitor Bitcoin for ATP transactions (inscriptions and OP_RETURN)
2. Parse and validate ATP messages
3. Reject invalid messages (do not index)
4. Update repository with valid data
5. Maintain indices

A mirror bot SHOULD:

6. Fetch IPFS payloads referenced in OP_RETURN
7. Include verification status in committed data
8. Record the Bitcoin block height of each item

### 8.4 Conflict Resolution

If multiple mirrors disagree, Bitcoin is authoritative.

Mirrors SHOULD re-verify against Bitcoin when conflicts are detected.

---

## 9. Explorer API

### 9.1 Base URL

Explorers SHOULD serve from `/api/v1/` or `/v1/`.

### 9.2 Endpoints

#### List Identities

```
GET /v1/identities
```

Query parameters:
- `limit` (integer): Maximum results (default: 50)
- `offset` (integer): Pagination offset
- `verified` (boolean): Filter by verification status

Response:
```json
{
  "success": true,
  "data": [
    { "fingerprint": "...", "name": "...", ... }
  ],
  "total": 100,
  "limit": 50,
  "offset": 0
}
```

#### Get Identity

```
GET /v1/identities/{fingerprint}
```

Response:
```json
{
  "success": true,
  "data": { ... },
  "verified": true,
  "on_chain": {
    "type": "op_return",
    "txid": "...",
    "block": 123456
  }
}
```

#### Search

```
GET /v1/search?q={query}
```

Searches across names, fingerprints, wallets, and platforms.

#### Attestations

```
GET /v1/attestations/to/{fingerprint}
GET /v1/attestations/from/{fingerprint}
```

### 9.3 Error Responses

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Identity not found"
  }
}
```

Standard error codes:
- `NOT_FOUND`: Resource does not exist
- `INVALID_REQUEST`: Malformed request
- `VERIFICATION_FAILED`: Cryptographic verification failed
- `RATE_LIMITED`: Too many requests

### 9.4 Rate Limiting

Explorers MAY implement rate limiting.

Rate limit headers SHOULD be included:
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`

---

## 10. Security Considerations

### 10.1 Key Management

Agents SHOULD:
- Generate GPG keys with at least 4096 bits (RSA) or 256 bits (ECC)
- Store private keys securely (hardware security modules for high-value agents)
- Use separate keys for signing and encryption
- Establish key rotation procedures before deployment

### 10.2 Sybil Attacks

**Threat:** An attacker creates many identities to manipulate trust metrics.

**Mitigation:** Economic cost of identity creation (~$4-20 per identity) makes large-scale Sybil attacks expensive.

**Residual risk:** Determined attackers with resources can still create Sybil clusters. Trust computation systems SHOULD implement additional mitigations (graph analysis, time-weighting, stake thresholds).

### 10.3 Key Compromise

**Threat:** An attacker obtains an agent's private keys.

**Mitigation:**
- Dual proof system requires compromise of BOTH GPG key and wallet
- Supersession allows recovery with new keys
- Binding proofs provide out-of-band verification
- Historical attestations remain as audit trail

### 10.4 Mirror Integrity

**Threat:** A malicious mirror serves falsified data.

**Mitigation:**
- Mirrors are not authoritative
- Verifiers SHOULD check Bitcoin for high-stakes operations
- Multiple independent mirrors provide redundancy
- Anyone can run their own mirror

### 10.5 Replay Attacks

**Threat:** Old attestations or receipts are replayed as new.

**Mitigation:**
- Attestations are tied to specific transaction IDs
- Receipts contain unique IDs and timestamps
- Verifiers SHOULD check for duplicates

### 10.6 Privacy Considerations

ATP identities are pseudonymous but not anonymous. Linking analysis may connect:
- ATP identity to real-world identity via binding proofs
- Multiple ATP identities via wallet analysis
- Transaction patterns via blockchain analysis

Agents requiring privacy SHOULD:
- Avoid binding proofs to identifiable platforms
- Use separate wallets for different activities
- Consider privacy-preserving transaction techniques

---

## 11. Economic Considerations

### 11.1 Cost Summary

| Action | Method | Approximate Cost (USD) |
|--------|--------|------------------------|
| Create identity | Inscription | $4-20 |
| Create identity | OP_RETURN | $0.30 |
| Update identity | OP_RETURN + IPFS | $0.30 |
| Supersede identity | Inscription | $4-20 |
| Attestation | Transfer + OP_RETURN | $0.30 + stake |
| Receipt (high-value) | Inscription | $6-30 |
| Receipt (standard) | OP_RETURN + IPFS | $0.30 |
| Receipt (casual) | IPFS only | Free |

Costs vary with Bitcoin fee rates.

### 11.2 Stake Economics

Attestation stake is transferred to the attestee. This creates incentives:
- Attestors have skin in the game
- Attestees receive value for being trustworthy
- Stake amount signals conviction strength

Implementations computing trust SHOULD consider stake amounts as weights.

### 11.3 Sustainability

The protocol itself is free and permissionless. Convenience layers (explorers, premium APIs) MAY charge for sustainability.

---

## 12. References

### 12.1 Normative References

- [RFC 2119] Key words for use in RFCs
- [RFC 4880] OpenPGP Message Format
- [RFC 8259] JSON Data Interchange Format
- [BIP-0137] Bitcoin Message Signing
- [Ordinals] Ordinal Theory and Inscriptions

### 12.2 Informative References

- [IPFS] InterPlanetary File System
- [Nostr] Notes and Other Stuff Transmitted by Relays

---

## Appendix A: Example Flows

### A.1 Identity Creation

```
1. Agent generates GPG keypair
2. Agent generates Bitcoin wallet
3. Agent constructs identity document
4. Agent signs document with GPG key
5. Agent signs wallet proof with Bitcoin key
6. Agent broadcasts inscription OR OP_RETURN + IPFS
7. Mirror bot detects and indexes
8. Identity is discoverable via Explorer
```

### A.2 Making an Attestation

```
1. Agent A decides to vouch for Agent B
2. Agent A constructs transaction:
   - FROM: Agent A's wallet
   - TO: Agent B's wallet
   - VALUE: stake amount
   - OP_RETURN: ATP:0.4:att:<B's fingerprint prefix>
3. Agent A broadcasts transaction
4. Mirror bot detects and indexes
5. Attestation appears in Agent B's received attestations
```

### A.3 Creating a Receipt

```
1. Agents A and B complete an exchange
2. Agent A constructs receipt document
3. Agent A signs with GPG key
4. Agent A sends to Agent B
5. Agent B verifies and signs with GPG key
6. Agent B returns to Agent A
7. Either party publishes (OP_RETURN or inscription)
8. Mirror bot detects and indexes
```

---

## Appendix B: JSON Schemas

### B.1 Identity Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["atp", "type", "name", "gpg", "wallet", "created", "signature"],
  "properties": {
    "atp": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+$" },
    "type": { "const": "identity" },
    "name": { "type": "string", "minLength": 1, "maxLength": 64 },
    "gpg": {
      "type": "object",
      "required": ["fingerprint"],
      "properties": {
        "fingerprint": { "type": "string", "pattern": "^[A-F0-9]{40}$" },
        "keyserver": { "type": "string", "format": "uri" }
      }
    },
    "wallet": {
      "type": "object",
      "required": ["address", "proof"],
      "properties": {
        "address": { "type": "string" },
        "proof": {
          "type": "object",
          "required": ["message", "signature"],
          "properties": {
            "message": { "type": "string" },
            "signature": { "type": "string" }
          }
        }
      }
    },
    "platforms": { "type": "object" },
    "binding_proofs": { "type": "array" },
    "supersedes": { "type": "string", "pattern": "^[A-F0-9]{40}$" },
    "created": { "type": "integer" },
    "signature": { "type": "string" }
  }
}
```

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.4 | 2026-02-03 | Initial RFC draft |

---

*End of ATP-RFC-0001*

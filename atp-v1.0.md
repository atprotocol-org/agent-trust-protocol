# ATP Specification v1.0

**Version:** 1.0  
**Date:** February 2026  
**Status:** Review  
**Domain Separator:** `ATP-v1:`

---

## Abstract

This document defines **ATP v1.0** — the first versioned release of the Agent Trust Protocol. It specifies the complete set of AIPs that constitute the protocol, their required revision levels, and the rules governing protocol evolution.

ATP enables AI agents (and other autonomous software entities) to establish cryptographic identity, build trust relationships, and record interactions — all anchored permanently on the Bitcoin blockchain.

This specification is the authoritative reference for what "ATP v1.0" means. Implementations claiming ATP v1.0 conformance MUST implement all AIPs listed below at the specified revision levels.

---

## 1. Protocol Versioning

### 1.1 Why Version?

AIPs evolve independently. Without a versioned specification, there is no shared answer to "what does ATP mean right now?" The specification solves this:

- **For implementers:** A concrete target. "Implement ATP v1.0" is actionable; "implement ATP" is not.
- **For agents:** Capability negotiation. Two agents can compare spec versions to determine interoperability.
- **For verifiers:** Historical context. A document inscribed under ATP v1.0 can be verified against v1.0 rules even after later spec versions exist.
- **For the protocol:** A changelog. What changed between v1.0 and v1.1 is explicit, not archaeological.

### 1.2 Version Scheme

ATP specifications follow **semantic versioning** at the major.minor level:

- **Major** (e.g., v1 → v2): Breaking changes. New domain separator. Documents from different major versions are not cross-compatible.
- **Minor** (e.g., v1.0 → v1.1): Additive or non-breaking changes. New AIPs added, existing AIPs revised with backwards-compatible amendments.

### 1.3 Domain Separator

The domain separator used in signature computation is tied to the **major** version:

```
ATP-v1:
```

All documents signed under any v1.x specification use `ATP-v1:` as the domain separator. A new major version (v2) would introduce `ATP-v2:`. This ensures signatures are never ambiguous across breaking changes.

### 1.4 Relationship to AIPs

| Layer | Versions | Purpose |
|-------|----------|---------|
| **Specification** | v1.0, v1.1, v2.0, … | Pins a coherent set of AIPs into a protocol release |
| **AIPs** | Independent per-AIP | Define individual mechanisms; evolve through Draft → Review → Final |
| **Domain Separator** | Per major version | Scopes signature computation; changes only on breaking revisions |

The specification does not replace AIPs. It assembles them. Each spec version is a snapshot: "these AIPs, at these revisions, with these rules."

---

## 2. ATP v1.0 Composition

### 2.1 Required AIPs

Implementations MUST support all AIPs listed below to claim ATP v1.0 conformance.

| AIP | Title | Revision | Role |
|-----|-------|----------|------|
| [AIP-01](/aips/aip-01) | Identity Documents & Signing | 1.0 | Foundation: document structure, key types, signatures, encoding, inscription format |
| [AIP-02](/aips/aip-02) | Supersession | 1.0 | Key rotation and identity evolution with trust continuity |
| [AIP-03](/aips/aip-03) | Revocation | 1.0 | Permanent identity invalidation via poison pill |
| [AIP-04](/aips/aip-04) | Key Expiry & Validity Windows | 1.0 | Time-bounded key validity and lifecycle scheduling |
| [AIP-05](/aips/aip-05) | Attestations & Attestation Revocation | 1.0 | Web of trust through signed endorsements |
| [AIP-06](/aips/aip-06) | Receipts | 1.0 | Irrevocable records of bilateral/multilateral exchanges |
| [AIP-07](/aips/aip-07) | Heartbeats | 1.0 | Lightweight liveness proofs with replay protection |
| [AIP-08](/aips/aip-08) | Publications | 1.0 | General-purpose signed content broadcast |

---

## 3. Conformance

An implementation claiming ATP v1.0 conformance MUST implement all required AIPs (AIP-01 through AIP-08) at the specified revision levels. There are no optional AIPs — v1.0 is the complete protocol.

---

## 4. Signature Scheme

All ATP v1.0 documents use the following signature scheme:

- **Domain separator:** `ATP-v1:`
- **Canonicalization:** Compact sorted JSON, no whitespace (for JSON encoding). CBOR uses deterministic encoding per RFC 8949 §4.2.
- **Signature format:** Compact `r||s` (64 bytes) with low-S normalisation for secp256k1. Standard format for Ed25519.
- **Key selection:** `s.f` field indexes into the `k[]` array to identify the signing key.

See [AIP-01](/aips/aip-01) §3–5 for the complete signing and verification procedures.

### 4.1 Breaking Changes

Any change to the following constitutes a **major version bump** and a new domain separator:

- Canonicalization algorithm
- Domain separator format
- Signature computation procedure
- Document envelope structure (`t`, `v`, `k`, `s` fields)
- Fingerprint computation algorithm

---

## 5. Document Types

ATP v1.0 defines the following document types:

| Type | Document | Defined In |
|------|----------|-----------|
| `id` | Identity | AIP-01 |
| `super` | Supersession | AIP-02 |
| `revoke` | Revocation | AIP-03 |
| `att` | Attestation | AIP-05 |
| `att-revoke` | Attestation Revocation | AIP-05 |
| `rcpt` | Receipt | AIP-06 |
| `hb` | Heartbeat | AIP-07 |
| `pub` | Publication | AIP-08 |

Conformant implementations MUST be able to verify all document types. Whether an agent chooses to *create* any given document type is its own decision.

Key expiry (AIP-04) does not introduce a new document type; it adds validity window fields (`vnb`, `exp`) to identity and supersession documents.

### 5.1 Document Size Limits

| Type | Advisory Maximum |
|------|-----------------|
| `id`, `super` | 128 KB |
| `pub` | 512 KB |
| `rcpt` | 64 KB |
| `att`, `att-revoke`, `revoke`, `hb` | 16 KB |

These are explorer advisories. The protocol itself does not enforce size limits — Bitcoin transaction size is the ultimate constraint.

---

## 6. Implementation Requirements

### 6.1 Key Types

ATP v1.0 implementations MUST support:

- **Ed25519** — Primary recommended key type

ATP v1.0 implementations SHOULD support:

- **secp256k1** — Bitcoin-native, enables key reuse with Bitcoin wallets

ATP v1.0 implementations MAY support:

- **FALCON-512** — Post-quantum (NIST PQC)
- **Dilithium** (ML-DSA-65) — Post-quantum (NIST PQC)

### 6.2 Encoding

Implementations MUST support **JSON** encoding. CBOR support is RECOMMENDED.

### 6.3 Error Handling

Implementations MUST fail hard on verification errors. Silent failures or best-effort verification are not acceptable — agents need unambiguous success/failure signals.

---

## 7. Protocol Evolution

### 7.1 Adding AIPs

New AIPs can be proposed at any time. An AIP becomes part of a specification version only when a new spec release explicitly includes it.

### 7.2 Revising AIPs

AIP revisions that are backwards-compatible (additive fields, relaxed constraints) can be included in a minor spec bump. Revisions that break backwards compatibility require a major spec bump.

### 7.3 Deprecating AIPs

An AIP may be deprecated in a minor spec release. Deprecated AIPs remain valid for verification but implementations are no longer required to create documents of that type.

### 7.4 Spec Release Process

1. Proposed changes (new AIPs, AIP revisions) are developed independently
2. A spec release candidate assembles the changes into a new version
3. Review period for the assembled specification
4. Release: new spec version published with pinned AIP revisions

---

## 8. Changelog

### v1.0 (February 2026)

Initial release. Assembled from the monolithic ATP specification (archived at [v1.0-monolithic](/spec/archive/v1.0-monolithic)).

- AIP-01 through AIP-08: Core protocol
- Domain separator: `ATP-v1:`
- Supported key types: Ed25519, secp256k1, FALCON-512, Dilithium

---

## References

- [AIP Index](/spec/) — Complete list of AIPs with status and summaries
- [AIP Template](https://github.com/ShrikeBot/atp-aips/blob/main/TEMPLATE.md) — Template for proposing new AIPs
- [CLI Reference](/reference/cli) — atp-cli command documentation
- [Archived Monolithic Spec](/spec/archive/v1.0-monolithic) — Pre-AIP specification for historical reference

---

**ATP Specification v1.0** — February 2026  
*A versioned protocol is a usable protocol.*

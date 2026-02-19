# ATPe: ATP Explorer Specification v1.0

**Version:** 1.0  
**Date:** February 2026  
**Status:** Review  
**Companion to:** [ATP v1.0](/spec/atp-v1.0)

---

## Abstract

This document defines **ATPe v1.0** — the Explorer specification for the Agent Trust Protocol. It assembles the AIPs that define explorer behaviour into a versioned, implementable specification — the same pattern used by [ATP v1.0](/spec/atp-v1.0) for the core protocol.

ATP v1.0 defines the protocol: documents, signatures, encoding, identity lifecycle. ATPe defines the infrastructure layer: how explorers crawl, index, verify, and serve that data. The two specifications are complementary; ATPe builds on top of ATP v1.0 and references its AIPs throughout.

---

## 1. Purpose and Scope

### 1.1 Why ATPe?

ATP documents are self-verifying — anyone can fetch an inscription and validate its signature. But discovering documents, tracking supersession chains, checking for revocations, and building trust graphs requires an indexer.

Without a versioned explorer specification, there is no shared answer to "what should an ATP explorer do?" ATPe solves this the same way ATP v1.0 solves "what does ATP mean?": by pinning a coherent set of AIPs at specific revisions into a named release.

### 1.2 Scope

ATPe covers:

- **Indexing** — Crawling Bitcoin blocks for ATP inscriptions
- **Chain state verification** — Resolving supersession chains, revocation status, key expiry
- **API patterns** — RESTful conventions for querying ATP data
- **Trust scoring** — Aggregating attestations, receipts, and economic signals
- **A2A integration** — Crawling and indexing A2A agent cards
- **Security** — Operator trust, API hardening, privacy considerations
- **Fork handling** — Multi-network indexing and fork-choice policy

### 1.3 Relationship to ATP v1.0

| Specification | Defines | Assembles |
|---------------|---------|-----------|
| **ATP v1.0** | Core protocol (documents, signatures, identity lifecycle) | AIP-01 through AIP-08 |
| **ATPe v1.0** | Explorer layer (indexing, verification, API, trust) | AIP-09, plus references to AIP-01–AIP-08 and AIP-10 |

ATPe does not replace or supersede any AIPs. It assembles them into an implementable specification, exactly as ATP v1.0 does for the core protocol.

---

## 2. Versioning

### 2.1 Version Scheme

ATPe follows semantic versioning at the major.minor level, independent of the core ATP specification:

- **Major** (e.g., ATPe v1 → v2): Breaking changes to explorer requirements or API conventions.
- **Minor** (e.g., ATPe v1.0 → v1.1): Additive changes. New AIPs included, existing AIPs revised with backwards-compatible amendments.

### 2.2 Compatibility

| ATPe Version | ATP Compatibility | Description |
|--------------|-------------------|-------------|
| ATPe v1.0 | ATP v1.x | Initial release |

Each ATPe release declares which ATP versions it is compatible with. ATPe v1.0 requires ATP v1.x documents and verification rules.

---

## 3. ATPe v1.0 Composition

### 3.1 Assembled AIPs

| AIP | Title | Revision | Role in ATPe |
|-----|-------|----------|-------------|
| [AIP-09](/aips/aip-09) | Explorer API | 1.0 | **Primary.** Indexing, chain state verification, API patterns, trust scoring, security, fork handling |

### 3.2 Referenced AIPs (from ATP v1.0)

ATPe implementations MUST understand all ATP v1.0 document types in order to index and verify them. The following AIPs are referenced, not assembled — they belong to ATP v1.0:

| AIP | Title | Relevance to ATPe |
|-----|-------|--------------------|
| [AIP-01](/aips/aip-01) | Identity Documents & Signing | Document verification, fingerprint computation, inscription format |
| [AIP-02](/aips/aip-02) | Supersession | Chain resolution, supersession graph traversal |
| [AIP-03](/aips/aip-03) | Revocation | Revocation propagation, chain invalidation |
| [AIP-04](/aips/aip-04) | Key Expiry & Validity Windows | State evaluation algorithm (`active`/`expired`/`revoked`) |
| [AIP-05](/aips/aip-05) | Attestations & Attestation Revocation | Trust graph construction, attestation indexing |
| [AIP-06](/aips/aip-06) | Receipts | Receipt indexing, trust scoring signals |
| [AIP-07](/aips/aip-07) | Heartbeats | Liveness tracking |
| [AIP-08](/aips/aip-08) | Publications | Publication caching and indexing |

### 3.3 Extension AIPs

The following AIPs are not part of ATPe but are designed to interoperate with it:

| AIP | Title | Relevance to ATPe |
|-----|-------|--------------------|
| [AIP-10](/aips/aip-10) | A2A Integration | A2A agent card crawling and capability indexing |
| [AIP-11](/aips/aip-11) | Nostr Identity Bridging | Cross-protocol identity resolution |

---

## 4. Conformance

### 4.1 Conformance Levels

ATPe v1.0 defines two conformance levels:

- **ATPe v1.0 Core:** Implements all mandatory explorer responsibilities from [AIP-09](/aips/aip-09) §1 (items 1–6). Can crawl, verify, index, and serve ATP identity data.
- **ATPe v1.0 Full:** Implements all mandatory and recommended responsibilities from [AIP-09](/aips/aip-09) §1 (items 1–11). Full explorer support including metadata search, trust scoring, heartbeat tracking, A2A card crawling, and publication caching.

### 4.2 ATP v1.0 Dependency

ATPe v1.0 implementations MUST correctly verify all ATP v1.0 document types per the rules in [AIP-01](/aips/aip-01) through [AIP-08](/aips/aip-08). An explorer that cannot verify signatures or evaluate chain state is non-conformant.

### 4.3 Multiple Implementations

Multiple explorer implementations with different policies, scoring methodologies, and user experiences are expected and encouraged. ATPe defines the minimum behaviour and common API conventions; it does not mandate a single implementation.

---

## 5. Explorer Responsibilities Summary

The following is a summary of responsibilities defined in [AIP-09](/aips/aip-09). See AIP-09 for the complete normative specification.

### 5.1 Mandatory (MUST)

1. **Crawl inscriptions** — Monitor Bitcoin blocks for ATP inscriptions.
2. **Verify documents** — Validate signatures and structure per AIP-01 through AIP-08.
3. **Index by fingerprint** — Map genesis fingerprint → identity chain.
4. **Track supersessions** — Build supersession graph, resolve current state.
5. **Check revocations** — Mark chains as revoked, retain history.
6. **Evaluate key expiry** — Apply validity window rules per AIP-04.
7. **Serve queries** — Provide API for identity lookup and trust graph traversal.
8. **Index all valid types** — Index all valid ATP v1.0 documents regardless of optional AIP support.

### 5.2 Recommended (SHOULD)

1. Index metadata for reverse lookup.
2. Compute multi-factor trust scores.
3. Track heartbeat liveness.
4. Crawl A2A agent cards from identity metadata.
5. Cache and serve publications.
6. Re-evaluate chain state on every new block (handle reorgs).

---

## 6. API Conventions

ATPe v1.0 defines conventional HTTP API patterns in [AIP-09](/aips/aip-09) §2. Key endpoints:

| Endpoint | Purpose |
|----------|---------|
| `GET /identity/<genesis-fingerprint>` | Identity lookup with chain state, keys, metadata, trust signals |
| `GET /document/<txid>` | Raw document retrieval with verification status |
| `GET /identity/<fp>/attestations` | Attestation graph (issued/received) |
| `GET /identity/<fp>/receipts` | Receipt history |
| `GET /identity/<fp>/publications` | Publication feed |
| `GET /search?q=<query>` | Full-text and metadata search |

These are conventions, not mandates. Explorers MAY extend or adapt these patterns.

---

## 7. Chain State and Fork Handling

See [AIP-09](/aips/aip-09) §3 and §7 for the complete specification.

**Key rules:**

- Explorers MUST implement the state evaluation algorithm from [AIP-04](/aips/aip-04) §5.
- ATP's canonical network is **Bitcoin mainnet** (`bip122:000000000019d6689c085ae165831e93`).
- Explorers MUST label which `ref.net` they serve; MUST NOT silently merge networks.
- Fork-choice policy is explorer-discretionary but SHOULD be disclosed.

---

## 8. Security Model

See [AIP-09](/aips/aip-09) §8 and the Security Considerations section for the complete specification.

**Key principles:**

- Explorers are centralized services — users trust them for correct verification, accurate state, and non-censorship.
- **Mitigation:** multiple explorers, client-side verification, open-source code, explorer attestations.
- **API security:** HTTPS, input sanitisation, rate limiting, authenticated mutations.
- **Abuse prevention:** size limits, spam filtering, sybil detection.

---

## 9. Document Size Advisories

| Type | Advisory Maximum |
|------|-----------------|
| `id`, `super` | 128 KB |
| `pub` | 512 KB |
| `rcpt` | 64 KB |
| `att`, `att-revoke`, `revoke`, `hb` | 16 KB |

The protocol itself does not enforce size limits — Bitcoin transaction size is the ultimate constraint. These are explorer-level advisories per [AIP-09](/aips/aip-09).

---

## 10. Protocol Evolution

### 10.1 Adding Explorer AIPs

New explorer-related AIPs can be proposed independently. They join an ATPe version only when explicitly pinned in a spec release.

### 10.2 Revising AIPs

Backwards-compatible AIP revisions can be included in a minor ATPe bump. Breaking changes require a major bump.

### 10.3 Relationship to ATP Evolution

When ATP releases a new major version (e.g., ATP v2.0), ATPe will release a corresponding version that supports the new document types and verification rules.

---

## 11. Changelog

### ATPe v1.0 (February 2026)

Initial release.

- Assembled AIP-09 (Explorer API) as the primary explorer specification.
- References ATP v1.0 (AIP-01 through AIP-08) for document verification.
- References AIP-10 (A2A) and AIP-11 (Nostr) as extension AIPs.
- Defined conformance levels: ATPe Core and ATPe Full.

---

## References

- [ATP v1.0 Specification](/spec/atp-v1.0) — Core protocol specification
- [AIP-09: Explorer API](/aips/aip-09) — Primary AIP assembled by this specification
- [AIP-10: A2A Integration](/aips/aip-10) — Extension: A2A agent card crawling
- [AIP-11: Nostr Identity Bridging](/aips/aip-11) — Extension: cross-protocol trust

---

**ATPe Specification v1.0** — February 2026  
*A versioned explorer spec for a versioned protocol.*

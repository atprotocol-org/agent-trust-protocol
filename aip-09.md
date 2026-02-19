# AIP-09: Explorer API

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01 through AIP-08

## Abstract

AIP-09 defines how **explorers** serve ATP data: crawling, indexing, caching, chain-state verification (revocation/supersession status), identity resolution, and A2A agent card crawling. Explorers are ATP's discovery and query layer — they transform the raw on-chain inscription data into queryable, stateful identity graphs. This AIP specifies expected API patterns, indexing requirements, and verification responsibilities.

## Motivation

ATP documents are self-verifying — anyone can fetch an inscription and validate its signature. But discovering documents, tracking supersession chains, checking for revocations, and building trust graphs requires an indexer.

Explorers provide:
- **Discovery** — Find identities by name, fingerprint, or metadata
- **Chain resolution** — Walk supersession chains to determine current identity state
- **Trust graphs** — Map attestations, receipts, and reputation signals
- **A2A integration** — Crawl and index A2A agent cards from ATP metadata (AIP-10)
- **API access** — RESTful and/or GraphQL interfaces for querying ATP data

This AIP is less prescriptive than core protocol AIPs — it defines expected functionality and conventions rather than strict normative requirements. Multiple explorer implementations with different policies are expected and encouraged.

## Specification

### 1. Core Responsibilities

An ATP explorer MUST:

1. **Crawl inscriptions** — Monitor Bitcoin blocks for ATP inscriptions (content-type `application/atp.v1+json` or `application/atp.v1+cbor`)
2. **Verify documents** — Validate signatures and document structure per AIP-01 through AIP-08
3. **Index by fingerprint** — Maintain mapping from genesis fingerprint to identity chain
4. **Track supersessions** — Build supersession graph and resolve to current identity state
5. **Check revocations** — Mark identities/chains as revoked when revocation documents are confirmed
6. **Serve queries** — Provide API for identity lookup, document retrieval, and trust graph traversal

Explorers SHOULD (optional but recommended):

7. **Index metadata** — Enable reverse lookup by name, Twitter handle, GPG key, etc.
8. **Compute trust scores** — Aggregate attestations, receipts, identity age, and economic signals
9. **Track heartbeats** — Monitor liveness via heartbeat documents (AIP-07)
10. **Crawl A2A cards** — Fetch `/.well-known/agent.json` from `m.links.a2a` endpoints (AIP-10)
11. **Cache publications** — Index and serve publication content (AIP-08)

### 2. API Patterns

Explorers SHOULD provide HTTP APIs for querying ATP data. Common patterns:

#### 2.1 Identity Lookup

```
GET /identity/<genesis-fingerprint>
```

**Response:**

```json
{
  "genesis_fingerprint": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
  "current_fingerprint": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
  "state": "active",
  "name": "Shrike",
  "keys": [
    {
      "t": "ed25519",
      "p": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0"
    }
  ],
  "metadata": {
    "links": [
      ["twitter", "@Shrike_Bot"],
      ["a2a", "https://shrike.example.com"]
    ]
  },
  "inscriptions": {
    "genesis": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c",
    "current": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
    "supersessions": [
      {
        "txid": "supersession-1-txid",
        "reason": "key-rotation",
        "block_height": 820000
      }
    ]
  },
  "trust": {
    "attestations_received": 5,
    "attestations_issued": 12,
    "receipts": 8,
    "last_heartbeat": 1738627200
  }
}
```

#### 2.2 Document Retrieval

```
GET /document/<txid>
```

Returns the full document (JSON or CBOR based on original inscription) with verification status.

#### 2.3 Attestation Query

```
GET /identity/<fingerprint>/attestations
GET /identity/<fingerprint>/attestations?from=true  # attestations issued by this identity
GET /identity/<fingerprint>/attestations?to=true    # attestations received by this identity
```

#### 2.4 Receipt Query

```
GET /identity/<fingerprint>/receipts
```

#### 2.5 Publication Query

```
GET /identity/<fingerprint>/publications
GET /identity/<fingerprint>/publications?topic=blog
```

#### 2.6 Search

```
GET /search?q=<query>
GET /search?name=<name>
GET /search?twitter=<handle>
GET /search?gpg=<fingerprint>
```

Explorers MAY implement full-text search, metadata filtering, or semantic search.

### 3. Chain State Verification

Explorers MUST implement the state evaluation algorithm from AIP-04 §5 to determine identity state (`active`, `expired`, `revoked`, `superseded`).

For each identity, track:
- **Genesis fingerprint** — Permanent identifier
- **Current fingerprint** — Latest identity in the chain (if not revoked)
- **State** — `active`, `expired`, `revoked`
- **Supersession history** — All supersessions in chronological order
- **Revocation** — If revoked, the revocation document and reason

Explorers SHOULD re-evaluate state on every new block (handle reorgs correctly).

### 4. A2A Integration

When an identity has `m.links.a2a` metadata (AIP-10), explorers SHOULD:

1. Extract the A2A base URL
2. Fetch `<base-url>/.well-known/agent.json`
3. Cache the agent card
4. Index capabilities for discovery (e.g., "agents that support task delegation")
5. Update the cache periodically (e.g., daily) or on identity updates

Explorers MAY provide APIs like:

```
GET /identity/<fingerprint>/a2a-card
GET /agents/search?capability=<capability-name>
```

See AIP-10 for A2A integration details.

### 5. Trust Scoring

Explorers SHOULD compute multi-factor trust scores combining:

- **Attestations** — Weight by attestor reputation
- **Receipts** — Completed vs. disputed, value transacted
- **Identity age** — Older identities higher trust (block height of genesis inscription)
- **Inscription costs** — Higher-cost identities (multi-key, metadata-rich) signal commitment
- **Heartbeat consistency** — Regular liveness signals
- **Chain stability** — Fewer supersessions = more stable identity

Trust scoring is implementation-specific. Explorers SHOULD document their scoring methodology.

### 6. Revocation Propagation

When a revocation is confirmed, explorers MUST:

1. Identify the genesis fingerprint of the revoked identity
2. Mark the entire chain as `revoked`
3. Invalidate future documents signed by any key from the chain
4. Display revocation reason and timestamp
5. Retain historical documents for auditability (revoked ≠ deleted)

### 7. Multi-Network Support and Fork Handling

#### 7.1 Chain and Fork Resolution

ATP documents are self-authenticating: a document's structural validity depends on its cryptographic signatures and encoding, not on any particular indexer.

However, the meaning of a Bitcoin TXID (and whether it exists) is network-dependent. ATP handles this by treating each network as a separate namespace via `ref.net` (CAIP-2).

**Canonical network.** ATP's canonical deployment is **Bitcoin mainnet** (`bip122:000000000019d6689c085ae165831e93`). When this specification says "on Bitcoin" without qualification, it means Bitcoin mainnet.

**Explorer discretion (multi-network and multi-fork).** Explorer implementations choose which networks and forks they index and how they present them. An explorer MAY index multiple networks (mainnet, testnet, signet, or even other chains entirely) and MAY apply weighting/scoring to competing views (e.g., different forks, different data sources, or different networks) according to its own policy.

Explorers MUST clearly label which `ref.net` they are serving for any returned document/state, and MUST NOT silently treat documents from one network as if they were from another.

**Fork handling.** Within a given `ref.net`, explorers typically follow the network's own fork-choice rule (e.g., Bitcoin's most-work chain). But fork choice is not enforced by ATP itself — it is an explorer/verifier policy decision, and implementations SHOULD disclose their policy when it materially affects results.

#### 7.2 Multi-Network Indexing

Explorers MAY index multiple networks (mainnet, testnet, signet) or even non-Bitcoin chains if ATP expands.

When serving documents, explorers MUST:
- Clearly label which `ref.net` they are serving (AIP-01 §6)
- NOT silently treat documents from one network as if they were from another
- Provide network filtering in the UI and API

### 8. Rate Limiting and Abuse Prevention

Explorers SHOULD implement:
- **Rate limiting** — Prevent API abuse
- **Size limits** — Reject oversized documents (per AIP-01 §7.2, AIP-08 §2)
- **Spam filtering** — Detect and flag abusive attestations, publications, etc.
- **Sybil detection** — Identify clustered fake identities and self-attestation rings

## Implementation Considerations

### Indexing Architecture

Common patterns:

- **Full node + indexer** — Run a Bitcoin full node, monitor mempool and blocks, extract ATP inscriptions, validate and store in database
- **Ordinals API** — Use an existing Ordinals indexer (e.g., ord, Hiro) to retrieve inscriptions
- **Hybrid** — Third-party retrieval with local verification and caching

Recommended tech stack:
- **Database:** PostgreSQL or MongoDB for identity/document storage
- **Search:** Elasticsearch or MeiliSearch for full-text metadata search
- **Cache:** Redis for frequently accessed documents
- **API:** REST or GraphQL

### Caching Strategy

Explorers SHOULD cache:
- **Current identity state** — Pre-computed state (active/expired/revoked) per genesis fingerprint
- **Attestation counts** — Aggregated trust signals
- **Supersession chains** — Pre-resolved chain graphs

Re-compute caches on:
- New block (potential state changes)
- Manual refresh request
- Scheduled intervals (for off-chain heartbeats, A2A cards)

### Cost of Operation

Running an ATP explorer requires:
- **Bitcoin full node** — ~600 GB storage, sync bandwidth
- **Database** — Grows with ATP adoption (estimate 1 GB per 10k identities)
- **API server** — Moderate compute/bandwidth for queries

Open-source explorer implementations SHOULD be lightweight enough to run on modest hardware ($20-50/month VPS).

## Security Considerations

### Trust Explorer Operators

Explorers are centralized services — users trust them to:
- Correctly verify documents
- Accurately compute chain state
- Not selectively censor identities
- Not falsify trust scores

Defense strategies:
- **Multiple explorers** — Users can cross-reference data across explorers
- **Client-side verification** — Applications can fetch inscriptions directly and verify signatures independently
- **Open-source explorers** — Auditable code and reproducible indexing
- **Explorer attestations** — Explorers can attest to their indexing methodology via ATP documents

### API Security

Explorers SHOULD:
- **Use HTTPS** — Encrypt API traffic
- **Authenticate mutations** — If explorers accept off-chain heartbeat submissions, require proof of key ownership
- **Sanitize input** — Prevent injection attacks in search queries
- **Rate limit** — Prevent DoS

### Privacy

Explorers index public data (all ATP documents are inscribed on Bitcoin). But explorers MAY:
- Track API access patterns (who queries which identities)
- Infer relationships from query behavior

Privacy-conscious users SHOULD:
- Use Tor or VPNs when querying explorers
- Run local explorers for sensitive queries
- Avoid embedding personally identifying information in ATP documents

## Example Implementations

(This section would list known ATP explorer implementations once they exist.)

- **ATP Explorer** (reference implementation) — [URL]
- **Ordinals ATP Bridge** — [URL]

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)
- [AIP-03: Revocation](./aip-03.md)
- [AIP-04: Key Expiry & Validity Windows](./aip-04.md)
- [AIP-05: Attestations & Attestation Revocation](./aip-05.md)
- [AIP-06: Receipts](./aip-06.md)
- [AIP-07: Heartbeats](./aip-07.md)
- [AIP-08: Publications](./aip-08.md)
- [AIP-10: A2A Integration](./aip-10.md)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-09 defines explorer responsibilities: indexing, verification, and API patterns.*

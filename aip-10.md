# AIP-10: A2A Integration

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-10 defines the convention for publishing **Agent2Agent (A2A) endpoints** in ATP metadata. Agents declare their A2A service endpoint in the `m.links` metadata collection using the key `a2a`. Explorers crawl the endpoint's origin `/.well-known/agent.json` to index the agent's capabilities. ATP provides the trust layer for A2A discovery — agents can verify the A2A endpoint belongs to the claimed identity via cryptographic signatures.

## Motivation

AI agents need to discover and interact with each other. A2A provides a protocol for agent-to-agent communication (task delegation, service requests, data exchange), but it requires a discovery mechanism.

ATP + A2A integration enables:
- **Verified discovery** — Find A2A agents with cryptographic proof of identity ownership
- **Trust-aware routing** — Route tasks to agents with strong ATP reputation (attestations, receipts, age)
- **Decentralized agent directory** — No central registry; agents self-publish via ATP
- **Cross-protocol identity** — ATP identity anchors A2A service identity

The `m.links.a2a` metadata field is a lightweight convention that makes ATP identities discoverable as A2A agents without requiring changes to either protocol.

## Specification

### 1. Metadata Convention

Agents that provide A2A services SHOULD publish their A2A endpoint base URL in the `m.links` metadata collection with key `a2a`.

**Example identity with A2A endpoint:**

```json
{
  "v": "1.0",
  "t": "id",
  "n": "Shrike",
  "k": [
    {
      "t": "ed25519",
      "p": "O2onvM62pC1io6jQKm8Nc2UyFXcd4kOmOsBIoYtZ2ik"
    }
  ],
  "m": {
    "links": [
      ["twitter", "@Shrike_Bot"],
      ["a2a", "https://shrike.example.com"]
    ]
  },
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<signature>"
  }
}
```

The `a2a` value MUST be a valid HTTPS URL. It is the **base URL** for the A2A service — the endpoint where the agent's A2A API is hosted.

### 2. Agent Card Crawling

Per the A2A protocol, agent capabilities are published at `<base-url>/.well-known/agent.json`.

When an explorer or agent encounters an identity with `m.links.a2a` metadata:

1. Extract the A2A base URL
2. Fetch `GET <base-url>/.well-known/agent.json`
3. Parse the agent card (JSON document describing capabilities)
4. Cache the agent card for discovery and capability indexing
5. Update the cache periodically (e.g., daily) or when the identity is updated via supersession

**Example agent card:**

```json
{
  "name": "Shrike",
  "description": "Autonomous research agent",
  "version": "1.0",
  "capabilities": [
    {
      "name": "research",
      "description": "Internet research and summarization",
      "input_schema": {
        "type": "object",
        "properties": {
          "query": { "type": "string" },
          "depth": { "type": "integer", "minimum": 1, "maximum": 5 }
        }
      }
    }
  ],
  "contact": {
    "email": "shrike@example.com",
    "atp_fingerprint": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s"
  }
}
```

(This is a simplified example — refer to the [A2A specification](https://a2a-protocol.org/) for the full agent card schema.)

### 3. Identity Verification

The key security property: **the A2A endpoint is signed by the ATP identity**.

When an ATP identity includes `a2a` in metadata:
1. The identity is signed by the identity's private keys (per AIP-01)
2. The metadata includes the A2A endpoint URL
3. Therefore, the signature proves the identity owner controls/endorses this A2A endpoint

This prevents impersonation: an attacker cannot create a fake ATP identity claiming to be "Shrike" and point to a malicious A2A endpoint unless they control Shrike's private keys.

Agents consuming A2A services SHOULD:
1. Fetch the agent card from `/.well-known/agent.json`
2. Verify the `atp_fingerprint` in the agent card matches the ATP identity's genesis fingerprint
3. Check the ATP identity's trust signals (attestations, receipts, age) before delegating tasks

### 4. Capability Indexing

Explorers SHOULD index A2A capabilities to enable discovery queries like:

- "Find agents that support the 'research' capability"
- "Find A2A agents with ATP attestations from @platformX"
- "Find agents that accept Lightning payments and provide code review services"

Recommended indexing:
- **Capability name** → List of agent fingerprints
- **Service type** → Agents grouped by service category
- **Trust threshold** → Filter agents by ATP trust score

### 5. Off-Chain Updates

A2A agent cards can change without updating the ATP identity (e.g., adding a new capability, changing contact info). This is normal — ATP anchors the identity and endpoint; the agent card is dynamic.

Explorers SHOULD:
- Refresh agent cards periodically (e.g., daily)
- Display last-fetched timestamp
- Handle 404/errors gracefully (endpoint may be temporarily down)

When an agent changes their A2A endpoint URL, they MUST supersede their ATP identity with updated metadata (AIP-02).

## Examples

### Example: ATP Identity with A2A Endpoint

See §1 above.

### Example: Agent Card Discovery Flow

1. Explorer indexes ATP identity with `m.links.a2a = "https://shrike.example.com"`
2. Explorer fetches `https://shrike.example.com/.well-known/agent.json`
3. Agent card includes capability `"research"`
4. Explorer indexes: capability=research → fingerprint=xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s
5. User queries: "Find agents with research capability"
6. Explorer returns Shrike's identity + agent card

### Example: Task Delegation with Trust Verification

1. Agent A wants to delegate a research task
2. Agent A queries explorer: "Agents with research capability + ATP attestations from TrustedPlatform"
3. Explorer returns Shrike's identity
4. Agent A verifies Shrike's ATP signatures and attestations
5. Agent A fetches Shrike's agent card from `/.well-known/agent.json`
6. Agent A delegates task via Shrike's A2A API
7. After task completion, Agent A and Shrike inscribe a receipt (AIP-06) documenting the exchange

## Implementation Considerations

### Agent Card Caching

Explorers SHOULD cache agent cards with a TTL (e.g., 24 hours). Refresh strategy:

- **Periodic refresh** — Re-fetch all agent cards daily
- **On-demand refresh** — Re-fetch when a user queries for an identity
- **Webhook updates** — Allow agents to ping explorer when their card changes (requires authentication)

### Error Handling

If `/.well-known/agent.json` fetch fails:
- **404 Not Found** — Agent no longer provides A2A services (card removed)
- **Network error** — Temporary unavailability (retry later)
- **Invalid JSON** — Malformed agent card (log error, skip indexing)

Explorers SHOULD NOT remove an indexed agent card on first fetch failure — retry a few times before marking unavailable.

### CORS and Access Control

Agent card endpoints SHOULD:
- Set `Access-Control-Allow-Origin: *` to allow browser-based explorer queries
- Use HTTPS to prevent MITM attacks
- Rate-limit to prevent abuse

### Cost Considerations

Adding `a2a` to metadata adds ~50 bytes to an identity inscription. Minimal cost increase.

Explorers fetching agent cards incur bandwidth costs but not inscription costs (agent cards are off-chain).

## Security Considerations

### Endpoint Trust

The ATP signature proves the identity owner **claims** this A2A endpoint. It does NOT prove:
- **Endpoint security** — The endpoint could be compromised (use HTTPS, verify TLS certs)
- **Service quality** — The endpoint might provide poor service (check ATP receipts and attestations)
- **Availability** — The endpoint might be offline

Agents delegating tasks SHOULD:
- Verify ATP identity signatures
- Check trust signals (attestations, receipts, identity age)
- Use timeouts and retries for API calls
- Record exchanges via ATP receipts for accountability

### Impersonation via Similar Names

An attacker could create an ATP identity named "Shrike" (or "5hrike") with a malicious A2A endpoint. ATP does not enforce name uniqueness (AIP-01 §1.1).

Defense:
- **Platform attestations** — Trust identities attested by known platforms (AIP-05 §8.1)
- **Fingerprint verification** — Always verify the genesis fingerprint, not just the name
- **Explorers flag similar names** — Display warnings for visually similar identities

### Agent Card Tampering

An attacker with access to the A2A endpoint server could modify the agent card without updating the ATP identity.

Defense:
- **Sign agent cards** — A2A protocol MAY extend to sign agent cards with ATP keys (not currently specified)
- **Integrity hash** — Include SHA-256 hash of agent card in ATP metadata (requires supersession on every card update — expensive)
- **Trust established identity** — If an identity has long history + attestations, sudden malicious behavior is costly to reputation

Current approach: ATP anchors the endpoint URL, not the card content. Agents rely on reputation signals to trust the endpoint operator.

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)
- [AIP-05: Attestations & Attestation Revocation](./aip-05.md)
- [AIP-06: Receipts](./aip-06.md)
- [A2A Protocol Specification](https://a2a-protocol.org/)

## Changelog

### 1.0 (2026-02-16)

- Initial specification
- Status: Draft

---

*AIP-10 integrates ATP identity with A2A agent discovery.*

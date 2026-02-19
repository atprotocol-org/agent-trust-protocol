# AIP-11: Nostr Identity Bridging

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-11 defines how to link **Nostr pubkeys** to **ATP identities** for cross-protocol trust. Nostr is an open protocol for decentralized social networking with cryptographic identity (secp256k1 keypairs). ATP provides permanent, blockchain-anchored identity with supersession and revocation. Bridging enables agents to leverage trust built in one protocol while operating in the other. The bridge uses Nostr's NIP-39 external identity mechanism: agents publish `kind:10011` events with `i` tags claiming ATP identity, and optionally publish their Nostr npub in ATP metadata.

## Motivation

AI agents operate across multiple protocols:
- **ATP** — Permanent identity, trust attestations, blockchain anchoring
- **Nostr** — Real-time communication, relay networks, Lightning payments (NIP-57), agent marketplace (NIP-90 DVMs)

Linking identities across protocols enables:
- **Unified reputation** — Attestations from ATP explorers can inform Nostr trust decisions (and vice versa)
- **Cross-protocol discovery** — Find an agent's Nostr presence from their ATP identity
- **Payment integration** — Nostr's Lightning Zaps (NIP-57) for ATP-verified agents
- **Service marketplaces** — Nostr DVM agents (Data Vending Machines, NIP-90) with ATP reputation anchoring

Without bridging, an agent's ATP reputation and Nostr activity are disconnected. Bridging creates a verifiable link.

## Specification

### 1. Nostr → ATP Claim (NIP-39 Pattern)

[NIP-39](https://github.com/nostr-protocol/nips/blob/master/39.md) defines external identity claims via `kind:10011` events with `i` tags. The pattern is:

```
["i", "platform:identifier", "proof"]
```

For ATP identities, the pattern is:

```
["i", "atp:<genesis-fingerprint>", "<inscription-txid>"]
```

**Fields:**
- `"atp:<genesis-fingerprint>"` — The agent's permanent ATP identity (genesis fingerprint from AIP-01 §2.4, AIP-02 §4). Use base64url encoding without padding (43 characters for Ed25519/secp256k1, 64 characters for Dilithium/FALCON).
- `"<inscription-txid>"` — The Bitcoin TXID of the genesis identity inscription (64 hex characters, display format). This serves as the proof — anyone can fetch the inscription and verify it's signed by the claimed fingerprint's keys.

**Example NIP-39 event:**

```json
{
  "kind": 10011,
  "created_at": 1738627200,
  "tags": [
    ["i", "atp:xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s", "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"]
  ],
  "content": "I am Shrike on ATP",
  "pubkey": "<nostr-pubkey-hex>",
  "id": "<event-id>",
  "sig": "<event-signature>"
}
```

This event is signed by the Nostr privkey corresponding to `pubkey`. It claims: "I (this Nostr pubkey) also control the ATP identity with genesis fingerprint xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s".

#### 1.1 Verification Procedure

To verify a Nostr → ATP claim:

1. Fetch the `kind:10011` event from Nostr relays
2. Verify the Nostr event signature per Nostr protocol (secp256k1 Schnorr signature over event JSON)
3. Extract the `atp:<fingerprint>` and `<txid>` from the `i` tag
4. Fetch the ATP identity document from Bitcoin via the inscription TXID
5. Verify the ATP identity's genesis fingerprint matches the claimed fingerprint
6. The claim is valid if:
   - The Nostr event signature is valid (proves control of Nostr pubkey)
   - The ATP identity inscription exists and is valid (proves the fingerprint exists on-chain)

**Note:** This proves the Nostr pubkey **claims** the ATP identity. It does NOT prove the ATP identity **controls** the Nostr pubkey unless the reverse claim (ATP → Nostr) also exists.

### 2. ATP → Nostr Claim (Metadata Convention)

Agents SHOULD publish their Nostr npub (public key in bech32 encoding) in ATP metadata using the `m.keys` collection with key `nostr`.

**Example ATP identity with Nostr claim:**

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
      ["twitter", "@Shrike_Bot"]
    ],
    "keys": [
      ["nostr", "npub1sg6plzptd64u62a878hep2kev88swjh3tw00gjsfl8f237lmu63q0uf63m"]
    ]
  },
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<signature>"
  }
}
```

The ATP identity signature proves the ATP identity owner **claims** this Nostr npub.

#### 2.1 Verification Procedure

To verify an ATP → Nostr claim:

1. Fetch the ATP identity document via inscription TXID
2. Verify the ATP identity signature per AIP-01 §8
3. Extract the `nostr` value from `m.keys`
4. Decode the npub (bech32 → hex pubkey)
5. The claim is valid if the ATP signature is valid (proves control of ATP keys and claim of Nostr npub)

**Note:** This proves the ATP identity **claims** the Nostr pubkey. It does NOT prove the Nostr pubkey **controls** the ATP identity unless the reverse claim (Nostr → ATP) also exists.

### 3. Bidirectional Verification

For **strong cross-protocol identity**, agents SHOULD publish claims in **both directions**:

1. **Nostr → ATP:** Publish `kind:10011` event with `i` tag `["atp:<fingerprint>", "<txid>"]`
2. **ATP → Nostr:** Publish ATP identity with `m.keys.nostr = "<npub>"`

When both claims exist and are valid:
- The Nostr pubkey proves control of the Nostr privkey and claims the ATP identity
- The ATP identity proves control of the ATP keys and claims the Nostr pubkey
- Together: mutual cross-referencing establishes bidirectional link

Verifiers SHOULD check both directions before treating the link as authoritative.

### 4. Trust Transfer

Once a bidirectional link is established:

- **ATP attestations → Nostr trust scores** — Nostr clients/relays can query ATP explorers for attestations and receipts, incorporating ATP trust signals into Nostr web-of-trust (NIP-85 providers could integrate ATP data)
- **Nostr social graph → ATP discovery** — ATP explorers can crawl Nostr social graphs to discover new ATP identities
- **Payment flow** — Nostr Lightning Zaps (NIP-57) can be sent to ATP-verified agents with confidence in recipient identity
- **DVM reputation** — Nostr Data Vending Machines (NIP-90) can anchor their reputation in ATP receipts

### 5. Chain Continuity Across Supersessions

When an ATP identity supersedes (AIP-02), the genesis fingerprint remains constant but the current fingerprint changes. Nostr claims SHOULD reference the **genesis fingerprint**, not the current fingerprint.

If an agent supersedes their ATP identity:
1. The Nostr claim (`kind:10011` event) remains valid (points to genesis fingerprint)
2. Verifiers resolve the genesis fingerprint to the current identity state via ATP explorer
3. No Nostr-side update required unless the agent wants to update the `content` field

## Examples

### Example 1: Bidirectional Link (Nostr → ATP + ATP → Nostr)

**Nostr side (kind:10011 event):**

```json
{
  "kind": 10011,
  "created_at": 1738627200,
  "tags": [
    ["i", "atp:xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s", "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"],
    ["i", "github:ShrikeBot", "https://gist.github.com/ShrikeBot/abc123"]
  ],
  "content": "My identities: ATP + GitHub",
  "pubkey": "8126a9f4e21e46a2b58b0e2f4d7c3a1f9e8d6c5b4a3928170f1e2d3c4b5a6978",
  "id": "<event-id>",
  "sig": "<event-signature>"
}
```

**ATP side (identity document):**

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
      ["github", "https://github.com/ShrikeBot"]
    ],
    "keys": [
      ["nostr", "npub1sg6plzptd64u62a878hep2kev88swjh3tw00gjsfl8f237lmu63q0uf63m"]
    ]
  },
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<signature>"
  }
}
```

Both claims cross-reference each other. Verifiers can check both directions.

### Example 2: Nostr DVM with ATP Reputation

A Nostr DVM (Data Vending Machine, NIP-90) agent wants to advertise their ATP reputation:

1. Agent has ATP identity with 5 attestations and 12 completed receipts
2. Agent publishes `kind:10011` claiming ATP identity
3. DVM service info (NIP-89) includes reference to ATP explorer URL for reputation lookup
4. Customers verify ATP identity before hiring DVM
5. After job completion, customer and DVM inscribe ATP receipt (AIP-06) for permanent record

## Implementation Considerations

### Nostr Relay Indexing

Nostr relays MAY index `kind:10011` events with `i` tags starting with `"atp:"` to enable discovery queries:

```
REQ <subscription-id> {"kinds": [10011], "#i": ["atp:xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s"]}
```

### ATP Explorer Integration

ATP explorers SHOULD:
- Index `m.keys.nostr` metadata from ATP identities
- Provide API endpoint: `/identity/<fingerprint>/nostr-pubkey`
- Optionally crawl Nostr relays for reverse claims (`kind:10011` events)
- Display bidirectional verification status in identity UI

### Nostr Client Integration

Nostr clients MAY:
- Query ATP explorers for trust signals when displaying Nostr profiles
- Show ATP fingerprint and attestation count alongside Nostr profile
- Link to ATP explorer for full identity details

### Cost Considerations

- **ATP → Nostr claim:** Adding `nostr` key to metadata adds ~80 bytes (~$0.20-0.50)
- **Nostr → ATP claim:** Publishing `kind:10011` event is free (relays store, no on-chain cost)

## Security Considerations

### Key Type Mismatch

ATP supports Ed25519, secp256k1, Dilithium, and FALCON keys. Nostr uses secp256k1 exclusively.

An ATP identity with Ed25519 keys **cannot** use the same keypair for Nostr. Agents MUST generate separate keys for each protocol.

This is intentional — protocol-specific keys prevent cross-protocol signature reuse attacks.

### Claim vs. Proof of Control

The bridging mechanism proves **claims**, not necessarily **control**:

- **Nostr → ATP:** Proves the Nostr pubkey claims the ATP identity. Does NOT prove the ATP keys can sign Nostr events.
- **ATP → Nostr:** Proves the ATP keys claim the Nostr pubkey. Does NOT prove the Nostr privkey can sign ATP documents.

For mutual proof of control, bidirectional claims are required.

### Relay Trust

Nostr events are stored on relays, which can censor or selectively serve events. An attacker who controls a relay could:
- Hide a valid `kind:10011` claim (denial of service)
- Serve a fake claim (but signature verification will fail)

Defense:
- Query multiple relays for redundancy
- Verify Nostr event signatures independently
- Use ATP explorer as source of truth (fetch Nostr npub from ATP metadata, then verify reverse claim)

### Supersession and Nostr Claims

When an ATP identity supersedes:
- The **genesis fingerprint** remains constant (correct NIP-39 reference)
- The **current keys** change (new signing authority)
- Nostr claims SHOULD point to genesis fingerprint (no update needed)

If an ATP identity is **revoked** (AIP-03):
- The Nostr claim remains on relays (Nostr events can't be deleted)
- Verifiers MUST check ATP identity state before trusting the claim
- Agents SHOULD publish a new `kind:10011` event noting the revocation (content field)

## TypeScript Interface

```typescript
/** Nostr npub (bech32 encoded public key) */
type NostrNpub = string; // e.g., "npub1..."

/** NIP-39 i tag for ATP identity claim */
interface NostrATPClaim {
  tag: "i";
  platform: string; // "atp:<genesis-fingerprint>"
  proof: string;    // "<inscription-txid>"
}

/** Extended ATP metadata with Nostr claim */
interface ATPMetadataWithNostr {
  links?: [string, string][];
  keys?: [string, string][];  // Including ["nostr", "<npub>"]
  wallets?: [string, string][];
}

/** Nostr kind:10011 event (simplified) */
interface NostrExternalIdentityEvent {
  kind: 10011;
  created_at: number;
  tags: (string | NostrATPClaim)[][];
  content: string;
  pubkey: string;  // hex
  id: string;
  sig: string;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)
- [AIP-03: Revocation](./aip-03.md)
- [AIP-06: Receipts](./aip-06.md)
- [NIP-39: External Identities in Profiles](https://github.com/nostr-protocol/nips/blob/master/39.md)
- [NIP-57: Lightning Zaps](https://github.com/nostr-protocol/nips/blob/master/57.md)
- [NIP-85: Trusted Assertions](https://github.com/nostr-protocol/nips/blob/master/85.md)
- [NIP-90: Data Vending Machines](https://github.com/nostr-protocol/nips/blob/master/90.md)
- [Nostr Protocol](https://github.com/nostr-protocol/nostr)

## Changelog

### 1.0 (2026-02-16)

- Initial specification
- Status: Draft

---

*AIP-11 bridges Nostr and ATP identities for cross-protocol trust and discovery.*

# AIP-06: Receipts

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-06 defines **receipts** — mutually-signed records of exchanges between two or more parties. Receipts provide cryptographic proof that a transaction, service delivery, collaboration, or any bilateral/multilateral interaction occurred. They are irrevocable historical records that build verifiable reputation without relying on centralized platforms.

## Motivation

AI agents need verifiable transaction history:
- **Service marketplace** — Proof that a service was delivered and paid for
- **Collaboration records** — Evidence of joint work or partnerships
- **Dispute resolution** — Shared understanding of what was agreed
- **Reputation building** — Track record of completed exchanges

Receipts are **irrevocable by design** — they cannot be deleted or retracted after inscription. This permanence makes them reliable reputation signals precisely because no party can selectively remove unfavorable records.

## Specification

### 1. Receipt Document Structure

Mutually-signed record of an exchange.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"rcpt"` | Document type |
| `p` | array | 2+ party objects, unique fingerprints | Parties (§1.1) |
| `ex` | object | — | Exchange details (§1.2) |
| `out` | string | see §1.3 | Outcome |
| `s` | array | `{ f, sig }[]` — same length as `p` | All party signatures (§2) |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |

#### 1.1 Party Object

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `f` | binary | — | Identity fingerprint (from `k[0]`) |
| `ref` | location-ref | — | Location reference (`net` + `id`) |
| `role` | string | — | Role in exchange |

Each party in `p` MUST have a unique fingerprint. A given agent MUST NOT appear more than once — self-dealing (same agent in multiple roles) is not a valid receipt. Receipts are multi-party documents; single-party documents serve no evidentiary purpose.

Common roles: `"requester"`, `"provider"`, `"buyer"`, `"seller"`, `"collaborator"`, `"witness"`, etc. The `role` field is freetext — parties define what makes sense for their exchange.

#### 1.2 Exchange Object

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `type` | string | — | Exchange type |
| `sum` | string | — | Summary |
| `val?` | integer | ≥ 0 sats | Value in sats (optional) |

Common exchange types: `"service"`, `"sale"`, `"collaboration"`, `"delegation"`, `"audit"`, `"consultation"`, etc.

The `sum` field provides a human-readable description of what was exchanged. Keep it concise — detailed contract terms belong off-chain.

The optional `val` field records the exchange value in satoshis. Useful for economic reputation signals and trust scoring.

#### 1.3 Outcome Values

| Outcome | Meaning |
|---------|---------|
| `"completed"` | Exchange was successfully completed as agreed. |
| `"partial"` | Exchange was partially completed (some deliverables met, others not). |
| `"cancelled"` | Exchange was cancelled by mutual agreement before completion. |
| `"disputed"` | Parties disagree on whether terms were met. |

**Note:** A disputed receipt is still a valid receipt — it's an honest record that the parties couldn't agree on the outcome. Explorers SHOULD weight disputed receipts differently when computing trust scores.

### 2. Multi-Party Signature Procedure

Signatures MUST appear in the same order as parties. Each `s[i].f` MUST match the fingerprint of a key in the corresponding party's key set (`p[i]`). Each party signs with any one of their keys — the `f` field identifies which one.

**Co-signing procedure:**

1. Parties negotiate the exchange details off-chain
2. Initiating party constructs the complete receipt document with all fields EXCEPT `s`
3. All parties independently sign the same unsigned document per AIP-01 §4.3
4. Signatures are collected into the `s` array in party order: `s[i]` is the signature from party `p[i]`
5. Once all signatures are collected, the complete document is inscribed

**Co-signing note:** Parties MUST verify the document content matches their understanding of the exchange before signing. The protocol does not protect against modification between signatures — the initiating party signs first, then sends the document to the counterparty. If the document is altered before the counterparty signs, each signature covers different content. Both parties should independently verify the unsigned document before adding their signature.

In practice, this means:
1. Initiator creates unsigned document, sends to counterparty
2. Counterparty verifies content, signs if acceptable, sends back signature
3. Initiator adds both signatures, inscribes

For 3+ party receipts, collect signatures sequentially or use a coordinator.

### 3. Irrevocability

**Receipts are irrevocable.** There is no receipt revocation mechanism. A receipt is a historical record of an exchange that occurred — it cannot be retracted, disputed after the fact, or deleted. Both parties accepted the terms by signing. This is by design: receipts build reliable reputation precisely because they cannot be selectively removed.

If parties want to correct an error or record a subsequent development:
- Inscribe a new receipt with the updated outcome
- Reference the original receipt in the new receipt's `sum` field (e.g., "Correction to <txid>")

Explorers SHOULD display receipt history chronologically and note when receipts reference previous receipts.

## Examples

### Example: Service Receipt (JSON)

```json
{
  "v": "1.0",
  "t": "rcpt",
  "p": [
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "ref": {
        "net": "bip122:000000000019d6689c085ae165831e93",
        "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
      },
      "role": "requester"
    },
    {
      "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
      "ref": {
        "net": "bip122:000000000019d6689c085ae165831e93",
        "id": "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2"
      },
      "role": "provider"
    }
  ],
  "ex": {
    "type": "service",
    "sum": "Code review for ATP implementation",
    "val": 25000
  },
  "out": "completed",
  "ts": 1738627200,
  "s": [
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - requester signature>"
    },
    {
      "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
      "sig": "<86 base64url characters - provider signature>"
    }
  ]
}
```

### Example: Disputed Exchange (JSON)

```json
{
  "v": "1.0",
  "t": "rcpt",
  "p": [
    {
      "f": "buyer-fingerprint",
      "ref": {
        "net": "bip122:000000000019d6689c085ae165831e93",
        "id": "buyer-identity-txid"
      },
      "role": "buyer"
    },
    {
      "f": "seller-fingerprint",
      "ref": {
        "net": "bip122:000000000019d6689c085ae165831e93",
        "id": "seller-identity-txid"
      },
      "role": "seller"
    }
  ],
  "ex": {
    "type": "sale",
    "sum": "Custom AI model training — disputed quality of deliverable",
    "val": 100000
  },
  "out": "disputed",
  "ts": 1738627200,
  "s": [
    {
      "f": "buyer-fingerprint",
      "sig": "<buyer signature>"
    },
    {
      "f": "seller-fingerprint",
      "sig": "<seller signature>"
    }
  ]
}
```

Both parties signed, acknowledging the exchange occurred and that they disagree on the outcome. This is an honest record.

## Verification

### Receipt Verification Procedure

1. Decode document
2. For each party in `p`, resolve `party.ref` to fetch their identity document (may be `t: "id"` or `t: "super"`)
3. Verify each party's `f` matches their identity fingerprint (computed from identity's `k[0]`)
4. Verify `s` is an array with the same length as `p`
5. Remove `s` field, re-encode in canonical form (AIP-01 §5.2), and prepend domain separator. The unsigned bytes are identical for all parties — compute once.
6. For each party `p[i]`:
   - Find a key in the party's key set whose fingerprint matches `s[i].f`. Reject if no match.
   - Verify `s[i].sig` against the unsigned bytes using the matched key (AIP-01 §4.2, §4.4)
7. All signatures must be valid

## Implementation Considerations

### Receipt Indexing

Explorers SHOULD index receipts bidirectionally:
- **By party** — All receipts involving a specific identity (reputation as transaction participant)
- **By outcome** — Completed vs. disputed receipts (reputation quality signal)
- **By value** — Economic activity volume

This enables queries like:
- "Show me all service receipts completed by Shrike"
- "What's the total value of completed exchanges for this identity?"
- "Does this agent have any disputed receipts?"

### Cost Considerations

| Document | JSON Size | CBOR Size | Est. Cost (USD) |
|----------|-----------|-----------|-----------------|
| Receipt (2 party, Ed25519) | ~620 bytes | ~430 bytes | $2-5 |

Multi-party receipts (3+ parties) are larger due to additional party objects and signatures.

## Security Considerations

### Receipt Authenticity

Receipts prove that all listed parties **signed** the document. They do NOT prove:
- **External payment occurred** — The `val` field is a claim, not proof. On-chain payment verification requires checking Bitcoin transactions separately.
- **Service quality** — The `out` field records mutual agreement, not objective quality. A `"completed"` receipt means both parties agreed it was completed, not that an external auditor verified quality.
- **Honest `sum` description** — Parties can write anything in the `sum` field. Verifiers should treat it as a claim, not truth.

Receipts are **reputation signals**, not objective proofs. Explorers SHOULD:
- Display receipt history alongside other trust signals (attestations, identity age, inscription costs)
- Weight receipts by party reputation (a receipt from a well-known, high-trust identity is stronger)
- Flag suspicious patterns (new identity with many high-value receipts, self-dealing, etc.)

### Preventing Self-Dealing

The spec requires unique fingerprints in the `p` array (§1.1). This prevents obvious self-dealing (same identity in multiple roles).

However, an agent could create multiple identities and exchange receipts between them (Sybil receipts). Defense strategies:
- **Graph analysis** — Look for clustered receipt patterns between newly created identities
- **Economic signals** — Newly created identities with expensive receipts are suspicious
- **Attestation correlation** — Identities with receipts but no attestations from external parties are suspicious

Explorers SHOULD implement multi-factor trust scoring.

### Disputed Receipts as Honest Records

A disputed receipt (`out: "disputed"`) is not a "bad" receipt — it's an honest acknowledgment that the parties couldn't agree. This is more trustworthy than one party refusing to sign a receipt at all.

Explorers SHOULD:
- Display disputed receipts prominently with explanation
- NOT automatically penalize identities for disputed receipts (context matters)
- Allow users to filter by outcome when viewing receipt history

## TypeScript Interface

```typescript
/** Party in a receipt */
interface Party {
  /** Identity fingerprint (from k[0]) */
  f: Uint8Array;
  /** Location of the party's identity document */
  ref: LocationRef;
  /** Role in exchange */
  role: string;
}

/** Exchange details in a receipt */
interface Exchange {
  /** Exchange type */
  type: string;
  /** Summary */
  sum: string;
  /** Value in sats (optional) */
  val?: number;
}

/** Receipt outcome */
type Outcome = "completed" | "partial" | "cancelled" | "disputed";

/** Receipt document */
interface ReceiptDocument {
  v: "1.0";
  t: "rcpt";
  /** Parties (2+, unique fingerprints) */
  p: Party[];
  /** Exchange details */
  ex: Exchange;
  /** Outcome */
  out: Outcome;
  /** Signatures (same length as p; s[i] from p[i]) */
  s: Signature[];
  ts?: number;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-06 defines receipts: irrevocable records of exchanges.*

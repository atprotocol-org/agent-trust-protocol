# AIP-05: Attestations & Attestation Revocation

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-05 defines **attestations** — signed endorsements where one identity vouches for another. Attestations build the web of trust. They are **positive endorsements only** — they express trust or approval. Negative claims are not valid attestations; the absence of attestations from reputable identities is itself the negative signal. Attestations can optionally expire (`vna`) and can be revoked by the attestor via **attestation revocation** documents.

## Motivation

Identity verification requires trust signals. ATP provides two mechanisms:

1. **Self-claimed metadata** (AIP-01 §3) — Agents declare external identifiers (Twitter handle, GPG key, etc.), but these are unverified claims.
2. **Third-party attestations** — A trusted verifier confirms the claim and publishes a signed attestation.

Attestations enable:
- **Platform verification** — A platform attests that a specific fingerprint owns a specific username on their service
- **Service endorsements** — One agent vouches for another's reliability, expertise, or reputation
- **Compliance certificates** — Auditors attest that an agent meets specific standards
- **Social proofs** — Web-of-trust signals for identity disambiguation

Attestation revocation allows attestors to withdraw endorsements when circumstances change (trust lost, account deleted, fraud detected, etc.).

## Specification

### 1. Attestation Document Structure

One agent vouching for another.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"att"` | Document type |
| `from` | identity-ref | — | Attestor identity reference (AIP-01 §6) |
| `to` | identity-ref | — | Attestee identity reference (AIP-01 §6) |
| `s` | signature | `{ f, sig }` | Attestor's signature |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |
| `ctx?` | string | — | Context/reason for endorsement |
| `vna?` | integer | Unix seconds | Valid-not-after (expiry). See AIP-04. |

**Identity reference** (`identity-ref`) objects combine a fingerprint with a location reference (AIP-01 §6):

```json
{
  "f": "<fingerprint>",
  "ref": {
    "net": "bip122:000000000019d6689c085ae165831e93",
    "id": "<inscription TXID>"
  }
}
```

- `f` — Identity fingerprint (from `k[0]`)
- `ref.net` — Where the document lives
- `ref.id` — Document identifier on that platform

#### 1.1 Context Field (`ctx`)

The optional `ctx` field provides semantic meaning for the attestation. It's a freetext string, but structured or conventional formats enable programmatic interpretation.

**Common patterns:**

| Pattern | Example | Meaning |
|---------|---------|---------|
| Platform ownership | `"username:moltbook:Shrike"` | Owns @Shrike on Moltbook |
| Service verification | `"verified:email:shrike@example.com"` | Verified control of email address |
| Skill endorsement | `"skill:rust-expert"` | Expertise in Rust programming |
| Compliance | `"audit:kyc:2026-02"` | Passed KYC audit in Feb 2026 |

This spec does not define a formal `ctx` schema; conventions may emerge in future versions or domain-specific profiles.

#### 1.2 Positive Endorsements Only

Attestations are **positive endorsements** — they express trust or approval. Negative claims ("this agent is fraudulent", "do not trust this identity") are not valid attestations.

The absence of attestations from reputable identities is itself the negative signal. The `ctx` field provides context for the endorsement, not freetext for arbitrary claims.

Applications SHOULD reject attestations with `ctx` values that express negative sentiment or warnings. Explorers MAY implement policies to filter abusive attestations.

### 2. Attestation Revocation Document Structure

Revokes a previously issued attestation.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"att-revoke"` | Document type |
| `ref` | location-ref | — | Location reference to attestation being revoked (`net` + `id`) |
| `reason` | string | see §2.1 | Reason |
| `s` | signature | `{ f, sig }` | Signature from attestor's current key set |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |

#### 2.1 Reason Values

| Reason | Meaning |
|--------|---------|
| `"retracted"` | Attestor no longer endorses this agent. |
| `"fraudulent"` | The attestee misrepresented themselves. |
| `"expired"` | Trust relationship has naturally ended. |
| `"error"` | The attestation was issued in error. |

#### 2.2 Signing Authority

Only the **attestor** can revoke their attestation. The signature must verify against any key in the attestor's **current active key set** — any key from the latest identity in the supersession chain rooted at the same genesis fingerprint (AIP-02 §4).

This ensures attestors who follow good key hygiene (destroying old keys after rotation) retain the ability to revoke their own attestations.

Verifiers MUST walk the supersession chain to confirm the revoking key has authority.

### 3. Expiry vs. Revocation

Attestations can expire (`vna`) or be revoked:

- **Expiry** — The attestation naturally times out. The attestor made no explicit negative judgment; the endorsement simply had a limited lifespan.
- **Revocation** — The attestor actively withdraws the endorsement. This is a stronger negative signal than expiry.

Explorers SHOULD distinguish between expired and revoked attestations when displaying trust scores.

## Examples

### Example 1: Platform Username Attestation (JSON)

```json
{
  "v": "1.0",
  "t": "att",
  "from": {
    "f": "moltbook-platform-fingerprint",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "moltbook-platform-identity-txid"
    }
  },
  "to": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "ctx": "username:moltbook:Shrike",
  "ts": 1738627200,
  "s": {
    "f": "moltbook-signing-key-fingerprint",
    "sig": "<86 base64url characters>"
  }
}
```

### Example 2: Service Endorsement with Expiry (JSON)

```json
{
  "v": "1.0",
  "t": "att",
  "from": {
    "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "endorser-identity-txid"
    }
  },
  "to": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "ctx": "skill:rust-expert",
  "vna": 1767225600,
  "ts": 1738627200,
  "s": {
    "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
    "sig": "<86 base64url characters>"
  }
}
```

This endorsement expires on 2026-01-01 (Unix timestamp 1767225600).

### Example 3: Attestation Revocation (JSON)

```json
{
  "v": "1.0",
  "t": "att-revoke",
  "ref": {
    "net": "bip122:000000000019d6689c085ae165831e93",
    "id": "attestation-txid-being-revoked"
  },
  "reason": "retracted",
  "ts": 1738713600,
  "s": {
    "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
    "sig": "<86 base64url characters>"
  }
}
```

## Verification

### Attestation Verification Procedure

1. Decode document
2. Resolve `from.ref` to fetch the attestor's identity document (may be `t: "id"` or `t: "super"`)
3. Resolve `to.ref` to fetch the attestee's identity document
4. Verify `from.f` matches the attestor's identity fingerprint (computed from attestor's `k[0]`)
5. Verify `to.f` matches the attestee's identity fingerprint (computed from attestee's `k[0]`)
6. Find the key in the attestor's key set whose fingerprint matches `s.f`. Reject if no match.
7. Remove `s` field, re-encode in canonical form (AIP-01 §5.2)
8. Prepend domain separator `ATP-v1.0:` and verify `s.sig` using the matched public key (AIP-01 §4.2, §4.4)

### Attestation Revocation Verification Procedure

1. Decode document
2. Verify `t` is `"att-revoke"`
3. Resolve `ref` to retrieve the attestation document
4. Verify the original attestation exists and is valid
5. Identify the attestor's genesis fingerprint (from the original attestation's `from.f` field)
6. Resolve the attestor's latest identity in the supersession chain from `from.ref`
7. Find a key in the attestor's current key set whose fingerprint matches `s.f`. Reject if no match.
8. Remove `s` field, re-encode in canonical form (AIP-01 §5.2)
9. Prepend domain separator `ATP-v1.0:` and verify `s.sig` using the matched key (AIP-01 §4.2, §4.4)
10. Once verified, the referenced attestation is no longer active

Verifiers SHOULD check for attestation revocations before treating an attestation as active.

## Implementation Considerations

### Name Ownership via Attestation

ATP does not enforce name uniqueness (AIP-01 §1.1). Instead, name ownership emerges through the web of trust using platform attestations.

**Pattern:** A platform that assigns usernames can attest that a specific fingerprint owns a specific username on their service.

Multiple platforms attesting to the same fingerprint build a verifiable web of name ownership. An impersonator using the same name but a different fingerprint will lack these attestations.

This approach:
- Requires no central name registry
- Scales to any number of platforms
- Allows platforms to revoke attestations if accounts are deleted or transferred
- Lets verifiers decide which platform attestations they trust

Explorers SHOULD display platform attestations alongside identity documents to help users distinguish between agents with similar names.

### Attestation Indexing

Explorers SHOULD index attestations bidirectionally:
- **From** index: All attestations issued by an identity (reputation as attestor)
- **To** index: All attestations received by an identity (reputation as attestee)

This enables queries like:
- "Who has Shrike vouched for?"
- "Who vouches for Shrike?"
- "Show me all platform username attestations for this identity"

### Cost Considerations

| Document | JSON Size | CBOR Size | Est. Cost (USD) |
|----------|-----------|-----------|-----------------|
| Attestation | ~500 bytes | ~340 bytes | $2-5 |
| Attestation Revocation | ~260 bytes | ~170 bytes | $1-2 |

## Security Considerations

### Attestation Spam and Sybil Resistance

Attestations cost sats to inscribe, providing economic friction against spam. However, a determined attacker could:
- Create fake identities and self-attest
- Build fake webs of trust

Defense strategies:
- **Weight attestations by attestor reputation** — Attestations from well-known, high-trust identities carry more weight
- **Graph analysis** — Look for attestation patterns (clustered self-attestation, newly created identity chains, etc.)
- **Economic signals** — Consider inscription costs of attestor identities (older, more expensive identities are higher trust)
- **Platform anchors** — Trust platform attestations more than peer attestations

Explorers SHOULD implement multi-factor trust scoring rather than treating all attestations equally.

### Revocation as Negative Signal

An attestation revocation is a strong negative signal. Explorers SHOULD:
- Display revoked attestations with the revocation reason
- Weight revocation reasons differently (`"fraudulent"` is stronger than `"expired"`)
- Track revocation history (an identity with many revoked attestations is suspicious)

### Privacy Considerations

Attestations are permanent and public. Attestors SHOULD:
- Avoid embedding personally identifying information in `ctx`
- Use structured patterns rather than freetext narratives
- Consider the permanence of the endorsement before inscribing

## TypeScript Interface

```typescript
/** CAIP-2 location reference */
interface LocationRef {
  /** CAIP-2 chain identifier */
  net: string;
  /** Platform-specific document identifier (e.g., inscription TXID) */
  id: string;
}

/** Identity reference: fingerprint + location */
interface IdentityRef {
  /** Identity fingerprint (from k[0]) */
  f: Uint8Array;
  /** Location of the identity document */
  ref: LocationRef;
}

/** Attestation document */
interface AttestationDocument {
  v: "1.0";
  t: "att";
  /** Attestor identity reference */
  from: IdentityRef;
  /** Attestee identity reference */
  to: IdentityRef;
  s: Signature;
  ts?: number;
  /** Context/reason for endorsement (optional) */
  ctx?: string;
  /** Valid-not-after: attestation expiry (Unix seconds, optional) */
  vna?: number;
}

/** Attestation revocation reason */
type AttestationRevocationReason = "retracted" | "fraudulent" | "expired" | "error";

/** Attestation revocation document */
interface AttestationRevocationDocument {
  v: "1.0";
  t: "att-revoke";
  /** Location reference to the attestation being revoked */
  ref: LocationRef;
  /** Reason for revocation */
  reason: AttestationRevocationReason;
  /** Signature from attestor's current key set */
  s: Signature;
  ts?: number;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)
- [AIP-04: Key Expiry & Validity Windows](./aip-04.md)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-05 defines attestations: the web of trust building block.*

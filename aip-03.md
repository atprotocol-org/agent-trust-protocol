# AIP-03: Revocation

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01, AIP-02

## Abstract

AIP-03 defines **revocation** — the permanent invalidation of an identity. Revocation is the nuclear option: once an identity is revoked, it's dead forever. No recovery, no supersession, nothing. The entire supersession chain is killed. Revocation uses a poison pill model where ANY key from ANY identity in the chain can trigger total death. This asymmetry — destroy but never take over — ensures that key compromise can never silently redirect an identity.

## Motivation

Identities need a death mechanism for two scenarios:

1. **Key compromise** — Private key may be in possession of an unauthorized party. Past actions should be treated with suspicion.
2. **Graceful shutdown** — Agent is permanently shutting down. Past actions remain trustworthy; no future actions expected.

Unlike supersession (which preserves trust continuity), revocation is terminal. An identity that's been revoked cannot be recovered. The owner must create a new identity from scratch.

The poison pill model serves as a dead man's switch: if a superseded key is later compromised, the owner (or the attacker) can still destroy the identity, but neither can take it over.

## Specification

### 1. Revocation Document Structure

Permanently invalidates an identity.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"revoke"` | Document type |
| `target` | identity-ref | — | Identity being revoked (`f` + `ref`) |
| `reason` | string | `"key-compromised"` \| `"defunct"` | Reason for revocation |
| `s` | signature | `{ f, sig }` | Signature from ANY key in the supersession chain |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |
| `vnb?` | integer | Unix seconds | Valid-not-before for this revocation (scheduled revocation). See AIP-04. |

#### 1.1 Reason Values

| Reason | Meaning |
|--------|---------|
| `"key-compromised"` | Private key may be in possession of an unauthorized party. Past actions should be treated with suspicion. |
| `"defunct"` | Agent is permanently shutting down. Past actions remain trustworthy; no future actions expected. |

### 2. Poison Pill Model

**Revocation kills the entire chain.** A revocation signed by any non-expired key from any identity in the supersession chain — current or historical — kills the ENTIRE identity chain. Not just from that key onwards: the whole thing. The identity is completely dead. The owner must create a new identity from scratch.

With multi-key identities, any single non-expired key from any identity in the chain can trigger revocation. If an identity held keys A, B, and C, any one of those keys can revoke (provided the key set has not expired — see AIP-04). This is a deliberate poison pill / dead man's switch:

- A coerced owner whose key was superseded away can nuke the hijacked identity using their old key (provided it has not expired).
- An attacker who cracks any single non-expired key from any point in the chain can destroy the identity but never take it over. Destruction is always possible; takeover never is.

A revocation is final. Once inscribed, the entire identity chain is permanently invalid. Verifiers MUST check for revocations against all non-expired keys from all identities in a supersession chain before accepting any identity as valid.

### 3. Signing Authority

The signature (`s.f`) MUST match the fingerprint of a key from:
- The **current** identity (latest in the supersession chain), OR
- Any **historical** identity in the chain (provided the key set has not expired)

Explorers/verifiers MUST walk the supersession chain backward to the genesis identity, collecting ALL keys from ALL identities in the chain (primary and secondary, current and historical) to build the **full key set**. A revocation is valid if signed by any non-expired key in this full key set.

### 4. Expiry Constraint

Keys from expired identity key sets cannot sign revocations (see AIP-04 §5.7.7). Once an identity's `vna` threshold has passed, its keys lose all authority — including revocation authority.

This limits the poison pill attack surface: historical keys can only revoke while they're still within their validity window. After expiry, they're harmless.

## Examples

### Example: Revocation (JSON)

```json
{
  "v": "1.0",
  "t": "revoke",
  "target": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "reason": "key-compromised",
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

### Example: Graceful Shutdown (JSON)

```json
{
  "v": "1.0",
  "t": "revoke",
  "target": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "reason": "defunct",
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

## Verification

### Revocation Verification Procedure

1. Decode document
2. Verify `t` is `"revoke"`
3. Resolve `target.ref` to fetch the identity document
4. Verify `target.f` matches the identity's fingerprint (computed from identity's `k[0]`)
5. Walk the supersession chain for the target identity backward to the genesis identity, collecting ALL keys from ALL identities in the chain (primary and secondary, current and historical) to build the **full key set**
6. Find the key in the full key set whose fingerprint matches `s.f`. Reject if no match.
7. Check that the key set containing the signing key has not expired (see AIP-04 for expiry evaluation). Reject if expired.
8. Remove `s` field, re-encode in canonical form (AIP-01 §5.2)
9. Prepend domain separator `ATP-v1.0:` and verify `s.sig` using the matched public key (AIP-01 §4.2, §4.4)
10. Once verified, the ENTIRE identity chain is permanently invalid — all identities linked by supersession

**Verifiers MUST check for revocations from ALL non-expired keys across ALL identities in a supersession chain before accepting any identity in that chain.**

## Implementation Considerations

### Explorers Must Track Revocations Globally

Revocations affect entire chains. When a revocation is confirmed:
1. Identify the genesis fingerprint of the target identity
2. Mark the entire chain (all identities linked by supersession) as `revoked`
3. Reject any future attestations, receipts, or heartbeats signed by keys from this chain

Recommended data model:
- Index revocations by genesis fingerprint
- Cache revocation status per chain
- Re-evaluate on reorgs

### Same-Block Revocation vs. Supersession

When a revocation and a supersession for the same identity are confirmed in the **same block**:

1. Process by **transaction position** within the block (earlier position first).
2. If position is identical (same transaction — unlikely but theoretically possible): **revocation takes precedence**. Destruction is always possible; recovery never is.

This deterministic ordering ensures all verifiers reach the same state.

### Cost Considerations

| Document | JSON Size | CBOR Size | Est. Cost (USD) |
|----------|-----------|-----------|-----------------|
| Revocation | ~280 bytes | ~180 bytes | $1-3 |

## Security Considerations

### Asymmetry: Destroy But Never Take Over

An attacker who compromises a historical key can:
- **Destroy** the identity via revocation (poison pill)
- **NOT take over** the identity (cannot supersede from a historical key)

The owner who still controls current keys can:
- **Supersede away** from a suspected compromise (preserves continuity)
- **Revoke** if supersession is too slow or impossible

This asymmetry prevents silent identity theft. Key compromise is always detectable (the attacker must either do nothing or destroy the identity).

### Chain Length as Attack Surface

Each supersession adds keys to the set that can revoke the entire chain. Multi-key identities amplify this: a chain of 3 identities with 2 keys each has 6 keys that can revoke (until keys expire).

Longer chains and more keys per identity mean a larger attack surface — an attacker who cracks any historical key can destroy (but not take over) the identity.

Agents should consider chain length and key count when evaluating their security posture. Mitigation strategies:
- Use `vna` to expire old key sets (AIP-04)
- Keep chains short (infrequent supersessions)
- Minimize keys per identity (only add keys when necessary)

### Proactive Revocation After Supersession

If you suspect an old key may be cracked AFTER you've already superseded away from it, proactively revoke using your CURRENT key. Because historical keys retain revocation authority (until they expire), an attacker who cracks a superseded key can destroy your identity even though you no longer use that key.

The proactive revocation strategy mitigates this by revoking from your current key BEFORE the old key is cracked. Once any key revokes, subsequent revocation attempts (from compromised historical keys) have no additional effect — the identity is already dead.

This is a scorched-earth defense: you destroy your own identity to prevent the attacker from using it. You must create a new identity from scratch, but at least the attacker gains nothing.

### Revocation vs. Expiry

Both are terminal states — the identity can no longer sign new documents. The difference is in how historical documents are interpreted:

- **Expiry** (AIP-04) means the authority window closed; historical documents signed before expiry remain fully trustworthy.
- **Revocation** means the keys may have been compromised; historical documents are suspect because the point of compromise may predate the revocation.

Explorers SHOULD treat expired identities differently from revoked identities when displaying historical attestations, receipts, and publications.

## TypeScript Interface

```typescript
/** Revocation reason */
type RevocationReason = "key-compromised" | "defunct";

/** Revocation document */
interface RevocationDocument {
  v: "1.0";
  t: "revoke";
  /** Identity being revoked */
  target: IdentityRef;
  /** Reason for revocation */
  reason: RevocationReason;
  /** Signature from ANY key in the supersession chain */
  s: Signature;
  ts?: number;
  /** Valid-not-before: scheduled revocation (Unix seconds, optional) */
  vnb?: number;
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

*AIP-03 defines the identity death mechanism: poison pill revocation.*

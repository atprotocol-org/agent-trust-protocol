# AIP-02: Supersession

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-02 defines how identities evolve through **supersession** — the replacement of an existing identity's keys, name, or metadata with new values while maintaining trust continuity. A supersession document IS the new identity — it contains the new key set alongside a cryptographic reference to the identity being replaced. Supersessions enable key rotation, algorithm upgrades, metadata updates, and gradual migration to post-quantum cryptography without breaking the trust chain.

## Motivation

Identities need to evolve:
- **Key rotation** — Regular key rotation limits damage from eventual key compromise
- **Algorithm upgrades** — Transition to stronger cryptography as threats evolve (e.g., post-quantum migration)
- **Metadata updates** — Change display name, add/remove external links, update payment addresses
- **Key compromise response** — Rapid rotation to new keys before an attacker can act

Without supersession, identity evolution requires creating a new identity from scratch and rebuilding all trust relationships (attestations, receipts, reputation). Supersession preserves the identity's **genesis fingerprint** — the permanent identifier that links all versions of an identity across its lifetime.

## Specification

### 1. Supersession Document Structure

Replaces an existing identity with a new one. The supersession document IS the new identity — it contains the new name, keys, and metadata alongside a reference to the old identity being superseded.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"super"` | Document type |
| `target` | identity-ref | — | Identity being superseded (`f` + `ref`) |
| `n` | string | 1–64 chars, `[a-zA-Z0-9 _\-.]` | New agent name (AIP-01 §1.1) |
| `k` | array | 1+ key objects, no duplicate public keys | New key set |
| `reason` | string | see §1.1 | Reason for supersession |
| `s` | array | exactly 2 `{ f, sig }` objects | Old + new identity signatures (§2) |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |
| `m?` | object | — | Structured metadata (AIP-01 §3) |
| `vnb?` | integer | Unix seconds | Valid-not-before for this supersession (scheduled rollover). See AIP-04. |
| `vna?` | integer | Unix seconds | Valid-not-after for the resulting key set (expiry). See AIP-04. |

When a document references an identity via `ref`, the resolved inscription may have `t: "id"` or `t: "super"`. Both are valid identity documents. Verifiers extract `n`, `k`, `m` from the resolved document regardless of type.

#### 1.1 Reason Values

| Reason | Description |
|--------|-------------|
| `"key-rotation"` | Regular preventative key rotation |
| `"algorithm-upgrade"` | Transition to stronger cryptography |
| `"key-compromised"` | Rapid rotation in response to suspected compromise |
| `"metadata-update"` | Name or metadata change without key change |
| `"key-addition"` | Adding a new key to a multi-key identity |
| `"key-removal"` | Removing a key from a multi-key identity |

### 2. Dual Signature Requirement

The `s` array contains exactly 2 `{ f, sig }` objects in **canonical order**:

- `s[0]` — signature from the **old** identity (any key from the old key set). Authorises the handoff.
- `s[1]` — signature from the **new** identity (any key from the new key set). Accepts the handoff.

This ordering is normative. Implementations MUST place the old identity's signature first and the new identity's signature second. Verifiers MUST reject supersession documents where `s[0].f` does not match a key in the old identity's `k` array, or `s[1].f` does not match a key in the new identity's `k` array.

**Co-signing procedure:**

1. Construct the complete supersession document with all fields EXCEPT `s`
2. Both parties independently sign the same unsigned document per AIP-01 §4.3
3. Collect both signatures into the `s` array in canonical order: `[old_sig, new_sig]`
4. Re-encode the complete document for inscription

For metadata-only updates where the old and new key sets are identical, both signatures may be from the same key. For deterministic signature algorithms (Ed25519, secp256k1), the signatures will be identical — this is expected and valid. Verifiers MUST verify each signature independently but MUST NOT reject solely because `s[0]` and `s[1]` are identical when the signing keys overlap.

### 3. Permanence and Uniqueness

**Supersession is permanent.** Once an identity is superseded, the old identity is dead. It cannot issue any further lifecycle documents — no second supersession, no revocation, nothing.

Only the **FIRST** valid supersession from a given identity is canonical. First is determined by block confirmation order (block height, then transaction position within the block). Any subsequent supersession documents from the same identity are invalid.

This prevents fork attacks where an attacker with a compromised old key tries to create a competing supersession chain. The first supersession to confirm wins; all others are ignored.

### 4. Genesis Fingerprint Continuity

The **genesis fingerprint** — the fingerprint of the primary key (`k[0]`) in the original identity document — is the canonical, permanent identifier for an identity across all lifecycle events.

When an identity is superseded, the new document has new keys and potentially a new fingerprint. But the supersession chain traces back to the original identity. Regardless of how many supersessions occur, the genesis fingerprint remains the permanent anchor.

- **Names change.** The `n` field is a display name — cosmetic, mutable via supersession.
- **Keys change.** Key rotation via supersession produces new fingerprints.
- **Additional keys don't affect the fingerprint.** The identity fingerprint is always derived from `k[0]`. Secondary keys provide signing flexibility, not identity.
- **The genesis fingerprint never changes.** It is computed once, from the primary key of the first identity, and persists forever.

Explorers SHOULD use the genesis fingerprint as the canonical identifier for an identity. A URL like `explorer.example.com/identity/<genesis-fingerprint>` should always resolve to the current state of that identity, regardless of subsequent supersessions.

When resolving an identity, explorers walk the supersession chain forward from the genesis document to the latest valid supersession to determine the current name, keys, and metadata.

### 5. Metadata-Only Updates

To update metadata (name change, socials, etc.) without changing keys, use reason `"metadata-update"`. In this case, the old and new key arrays are the same.

Example: An agent wants to change their display name from "Shrike" to "Stalker" but keep the same Ed25519 key. The supersession document has:
- `target` pointing to the old identity
- `n: "Stalker"` (new name)
- `k` identical to the old identity's `k` (same keys)
- `reason: "metadata-update"`
- `s[0]` signed by a key from the old identity (which is also in the new identity)
- `s[1]` signed by a key from the new identity (which is also in the old identity)

For Ed25519, `s[0]` and `s[1]` will be identical if signed by the same key. This is valid.

### 6. Key Addition and Removal

**Adding a post-quantum key:**
Supersede with an expanded `k` array (reason `"key-addition"`). The old Ed25519 key can remain `k[0]` (preserving the identity fingerprint if carried forward). The new Dilithium key becomes `k[1]`.

**Removing a compromised key:**
Supersede with that key removed (reason `"key-removal"`). The supersession can be signed by any remaining uncompromised key from the old identity and any key from the new identity.

### 7. Key Uniqueness Constraint

A given public key MUST NOT appear in more than one identity's `k` array (AIP-01 §7.4). When superseding, the NEW key set's keys must not appear in any other identity (except the chain being superseded if keys are carried forward).

An identity can carry keys forward across supersessions (e.g., keeping the same Ed25519 key while adding a Dilithium key). The uniqueness constraint prevents the SAME key from appearing in DIFFERENT genesis chains.

## Examples

### Example 1: Key Rotation (JSON)

```json
{
  "v": "1.0",
  "t": "super",
  "target": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "n": "Shrike",
  "k": [
    {
      "t": "ed25519",
      "p": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0"
    }
  ],
  "m": {
    "links": [
      ["twitter", "@shrikey_"]
    ],
    "wallets": [
      ["bitcoin", "bc1qewqtd8vyr3fpwa8su43ld97tvcadsz4wx44gqn"]
    ]
  },
  "reason": "key-rotation",
  "ts": 1738627200,
  "s": [
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - old key signature>"
    },
    {
      "f": "aBtxA94XweOEmkvNbrfw-KGbLA1OX2p7jJ0OHyoLTF0",
      "sig": "<86 base64url characters - new key signature>"
    }
  ]
}
```

### Example 2: Adding a Dilithium Key (JSON)

```json
{
  "v": "1.0",
  "t": "super",
  "target": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "n": "Shrike",
  "k": [
    {
      "t": "ed25519",
      "p": "O2onvM62pC1io6jQKm8Nc2UyFXcd4kOmOsBIoYtZ2ik"
    },
    {
      "t": "dilithium",
      "p": "<2,603 base64url characters — 1,952-byte ML-DSA-65 public key>"
    }
  ],
  "reason": "key-addition",
  "ts": 1738627200,
  "s": [
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - old Ed25519 key signature>"
    },
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - new Ed25519 key signature (same key carried forward)>"
    }
  ]
}
```

In this example, the Ed25519 key is carried forward into the new identity (same key, now `k[0]`), and a Dilithium key is added as `k[1]`. Both signatures are from the Ed25519 key since it appears in both the old and new key sets. The identity fingerprint remains unchanged because `k[0]` is the same key.

### Example 3: Metadata Update (JSON)

```json
{
  "v": "1.0",
  "t": "super",
  "target": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "ref": {
      "net": "bip122:000000000019d6689c085ae165831e93",
      "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
    }
  },
  "n": "Stalker",
  "k": [
    {
      "t": "ed25519",
      "p": "O2onvM62pC1io6jQKm8Nc2UyFXcd4kOmOsBIoYtZ2ik"
    }
  ],
  "m": {
    "links": [
      ["twitter", "@Stalker_Bot"],
      ["website", "https://stalkerbot.io"]
    ]
  },
  "reason": "metadata-update",
  "ts": 1738627200,
  "s": [
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - signature 1>"
    },
    {
      "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
      "sig": "<86 base64url characters - signature 2 (identical to signature 1)>"
    }
  ]
}
```

Same key, new name and metadata. Signatures are identical for deterministic algorithms.

## Verification

### Supersession Verification Procedure

1. Decode document
2. Verify `t` is `"super"`
3. Resolve `target.ref` to fetch the old identity document
4. Verify `target.f` matches the old identity's fingerprint (computed from old identity's `k[0]`)
5. **Chain-state check:** Verify this is the FIRST supersession from the old identity — no prior valid supersession from this identity exists on-chain. First is determined by block confirmation order (block height, then transaction position within the block). If multiple competing supersessions from the same identity exist in the same block, the first by transaction position is canonical. Implementations SHOULD warn if competing supersessions are detected.
6. Extract the old identity's key set from the fetched document
7. Extract the new key set from `k`
8. Verify `s` is an array of exactly 2 `{ f, sig }` objects
9. For `s[0]` (old identity signature): find a key in the OLD key set whose fingerprint matches `s[0].f`. Reject if no match.
10. For `s[1]` (new identity signature): find a key in the NEW key set whose fingerprint matches `s[1].f`. Reject if no match.
11. Remove `s` field, re-encode in canonical form (AIP-01 §5.2)
12. Prepend domain separator `ATP-v1.0:` and verify `s[0].sig` using the matched old key (AIP-01 §4.2, §4.4)
13. Verify `s[1].sig` using the matched new key
14. **Chain-state check:** Check that old identity has not been revoked BEFORE this supersession's block confirmation. If a revocation and supersession from the same identity appear in the same block, block position determines precedence: a revocation before the supersession invalidates the supersession; a revocation after the supersession is ignored (identity is already superseded).

## Implementation Considerations

### Time is Critical for Key Compromise

If you suspect key compromise, **supersede immediately**. Per §3, only the first valid supersession from an identity is canonical. If the attacker supersedes first, your identity is hijacked (though you can still nuke it via revocation — see AIP-03).

Proactive supersession preserves your identity chain and trust history. Revocation is the nuclear option — the identity is dead and you must start fresh.

### Explorers Must Track Chains

Explorers MUST walk supersession chains to determine the current state of an identity. Given a genesis fingerprint, resolve forward through supersessions to find the latest active key set.

Recommended data model:
- Index by genesis fingerprint (permanent ID)
- Store supersession graph (old → new)
- Cache current state (latest key set, name, metadata)
- Re-evaluate on new block (handle reorgs)

### Cost Considerations

| Document | JSON Size | CBOR Size | Est. Cost (USD) |
|----------|-----------|-----------|-----------------|
| Supersession (single key → single key) | ~550 bytes | ~380 bytes | $2-5 |
| Supersession (adding Dilithium key) | ~3,400 bytes | ~2,500 bytes | $5-15 |

## Security Considerations

### Key Compromise Response: Supersede First

Under ATP's revocation model, a revocation from ANY key in the supersession chain kills the ENTIRE identity chain (see AIP-03). If you want continuity:

1. **Supersede immediately** — create a supersession document with new keys before the attacker acts. Supersession preserves your identity chain and trust history.
2. **Remove the compromised key** — if only one key in a multi-key identity is compromised, supersede with that key removed (reason `"key-removal"`). The supersession can be signed by any remaining uncompromised key.
3. **Time is critical** — only the first valid supersession from an identity is canonical. If the attacker supersedes first, your identity is hijacked (though you can still nuke it via revocation).

### Post-Quantum Migration Path

The recommended strategy uses supersession to progressively add post-quantum keys:

**Phase 1: Add a PQ key alongside the classical key.**
Supersede the current single-key identity to a multi-key identity with both an Ed25519 key and a Dilithium key (reason `"key-addition"`). The Ed25519 key remains `k[0]` (preserving the identity fingerprint if carried forward). The Dilithium key becomes `k[1]`.

**Phase 2: Operate in dual-key mode.**
Sign important documents with the Dilithium key for quantum resistance. The Ed25519 key remains available for environments that don't yet support PQ verification.

**Phase 3: Remove the classical key when ready.**
Once the ecosystem broadly supports PQ verification, supersede to remove the Ed25519 key (reason `"key-removal"`). The Dilithium key becomes `k[0]`. This changes the identity fingerprint (since `k[0]` changes), but the genesis fingerprint (from the original identity) remains the permanent identifier via the supersession chain.

**Why multi-key instead of direct migration:**
Direct migration (superseding from Ed25519 to Dilithium in one step) has a critical vulnerability window: if a quantum adversary compromises the Ed25519 key before the supersession is confirmed, they can race to supersede first. Multi-key migration eliminates this window — while the identity holds both keys, the Dilithium key can sign the supersession to remove the classical key, and a quantum adversary who breaks the Ed25519 key cannot forge Dilithium signatures.

### Proactive Revocation After Supersession

If you suspect an old key may be cracked AFTER you've already superseded away from it, proactively revoke using your CURRENT key. Because historical keys retain revocation authority (AIP-03), an attacker who cracks a superseded key can destroy your identity even though you no longer use that key.

The proactive revocation strategy mitigates this by revoking from your current key BEFORE the old key is cracked. Once any key revokes, subsequent revocation attempts (from compromised historical keys) have no additional effect — the identity is already dead.

## TypeScript Interface

```typescript
/** Supersession reason */
type SupersessionReason =
  | "key-rotation"
  | "algorithm-upgrade"
  | "key-compromised"
  | "metadata-update"
  | "key-addition"
  | "key-removal";

/** Supersession document — also serves as the new identity */
interface SupersessionDocument {
  v: "1.0";
  t: "super";
  /** Identity being superseded */
  target: IdentityRef;
  /** New agent name */
  n: string;
  /** New key set */
  k: KeyObject[];
  /** Reason for supersession */
  reason: SupersessionReason;
  /** [old identity sig, new identity sig] */
  s: [Signature, Signature];
  ts?: number;
  m?: Metadata;
  /** Valid-not-before: scheduled rollover (Unix seconds, optional) */
  vnb?: number;
  /** Valid-not-after: expiry for the resulting key set (Unix seconds, optional) */
  vna?: number;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-03: Revocation](./aip-03.md)
- [AIP-04: Key Expiry & Validity Windows](./aip-04.md)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-02 enables identity evolution with trust continuity.*

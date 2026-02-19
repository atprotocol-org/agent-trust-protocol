# AIP-04: Key Expiry & Validity Windows

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-04 defines **validity windows** for identity key sets: `vna` (valid-not-after) for expiry and `vnb` (valid-not-before) for scheduled activation. Expiry is terminal — no supersession or revocation from expired keys. Expired ≠ revoked: expiry means the authority window closed (historical docs remain trustworthy), while revocation means keys may be compromised (historical docs are suspect). Validity evaluation uses Bitcoin Median Time Past (MTP) as the canonical time source.

## Motivation

Key sets need expiration for several reasons:

- **Limit exposure** — Keys with bounded lifespans reduce the damage from eventual compromise
- **Force rotation** — Scheduled expiry encourages regular key hygiene
- **Reduce poison pill attack surface** — Expired keys cannot trigger revocation (AIP-03), limiting the set of keys that can destroy an identity chain
- **Operational windows** — Temporary identities or time-limited delegated keys

Validity windows also enable **scheduled supersessions** and **scheduled revocations** — lifecycle events that activate at a specific chain time rather than immediately upon confirmation.

## Specification

### 1. Validity Window Fields

ATP supports optional validity windows on a small set of lifecycle documents:

- `vnb` (**valid-not-before**) — the document MUST NOT be treated as active until chain time is at or after this timestamp.
- `vna` (**valid-not-after**) — the identity key set MUST be treated as expired after this timestamp.

**Allowed usage:**

| Document Type | `vnb` Allowed | `vna` Allowed |
|---------------|---------------|---------------|
| Identity (`t: "id"`) | No | Yes |
| Supersession (`t: "super"`) | Yes | Yes |
| Revocation (`t: "revoke"`) | Yes | No |

For supersession documents:
- `vnb` applies to the **supersession itself** (scheduled rollover)
- `vna` applies to the **resulting key set** (the new identity created by the supersession)

### 2. Canonical Time Source: Bitcoin Median Time Past (MTP)

Because `ts` is creator-supplied, validity windows MUST be evaluated against a chain-derived time source.

For Bitcoin (`ref.net = bip122:000000000019d6689c085ae165831e93`), implementations MUST use **Median Time Past (MTP)** of the block that confirms the inscription.

**Definition (Bitcoin MTP):** For a block `B`, MTP is the median of the `time` fields of the last 11 block headers ending at `B` (i.e., `B`, `B-1`, ... `B-10`).

Explorers/verifiers MUST NOT fall back to local wall-clock time for `vnb`/`vna` evaluation. If chain time cannot be determined (e.g., missing block context), the document's active/expired status is **unknown**.

### 3. Identity States

An identity exists in exactly one of the following states at any given chain time:

| State | Meaning |
|-------|---------|
| **active** | Identity is valid and operational. Documents signed by its keys are accepted. |
| **expired** | `vna` threshold has passed. The identity's authority window has closed. No new documents of any kind may be signed by expired keys. Historical documents signed before expiry remain valid — expiry is not retroactive and does not imply compromise. |
| **revoked** | A revocation document has taken effect (AIP-03). The identity is permanently dead. No supersession, no recovery. The entire chain is poisoned. |
| **superseded** | A supersession has taken effect (AIP-02). The old key set is retired. The identity continues under new keys (follow the chain forward). |

**Note on `superseded`:** This state describes **intermediate identities in the chain** — prior key sets that have been replaced. When evaluating identity state, explorers walk the supersession chain to the tip and return the tip's state: `active`, `expired`, or `revoked`. The `superseded` state is never the terminal output, because the algorithm always resolves to the current chain head. Explorers SHOULD use `superseded` when displaying the state of historical key sets within a chain.

**Distinction: expired vs revoked:**
Both are terminal states — the identity can no longer sign new documents. The difference is in how historical documents are interpreted:
- **Expiry** means the authority window closed; historical documents signed before expiry remain fully trustworthy.
- **Revocation** means the keys may have been compromised; historical documents are suspect because the point of compromise may predate the revocation.

### 4. Document Activation Rules

Documents with `vnb` or `vna` follow these activation rules. Two time references are used:

- **`MTP(B)`** — the MTP of the chain tip block. Used for `vnb` activation checks ("has this document's activation threshold been reached?").
- **`MTP(B_D)`** — the MTP of the block that includes document `D`'s transaction. Used for authority checks ("was the signing identity still active when this document was inscribed?").

Rules:

**Supersession (`t: "super"`):**
- If `vnb` is present: the supersession is **pending** until `MTP(B) >= vnb`. Before that threshold, the old identity remains in its current state.
- If `vna` is present: the `vna` applies to the **new** identity created by the supersession (not the old one). The new key set expires when `MTP(B) > vna`.
- A supersession without `vnb` is **immediately active** upon block confirmation.

**Revocation (`t: "revoke"`):**
- If `vnb` is present: the revocation is **pending** until `MTP(B) >= vnb`.
- A revocation without `vnb` is **immediately active** upon block confirmation.

**Identity (`t: "id"`):**
- If `vna` is present: the identity expires when `MTP(B) > vna`.

### 5. State Evaluation Algorithm

Given an identity's genesis fingerprint, an evaluator computes the current state by processing all lifecycle documents in **strict block order**. Within a single block, documents are ordered by transaction position (miner-determined).

**Input:** All confirmed lifecycle documents (`id`, `super`, `revoke`) referencing this identity chain, plus a chain tip block `B` (whose MTP determines the current time).

**Procedure:**

1. **Initialise.** Set `current_keys` to the genesis identity's `k`. Set `state = active`. Set `vna_threshold = null`. If the genesis identity has `vna`, set `vna_threshold = vna`.

2. **Collect.** Starting from the genesis identity, gather all `super` and `revoke` documents whose `target.f` matches any identity fingerprint in the chain (i.e., the primary fingerprint of each successive identity, walking forward from genesis through supersessions). Sort by `(block_height, tx_position)` ascending.

3. **Process each document in order.** For each document `D` included in block `B_D`:

   **a) If `D` is a revocation (`t: "revoke"`):**
   - If `state = revoked`: skip (already dead).
   - If `vna_threshold` is not null and `MTP(B_D) > vna_threshold`: skip (identity was expired when this revocation was inscribed).
   - If `D.vnb` is present and `MTP(B) < D.vnb`: record as **pending revocation**; skip for now.
   - Otherwise: set `state = revoked`. Record `D.reason`. Stop processing — no further documents matter.

   **b) If `D` is a supersession (`t: "super"`):**
   - If `state = revoked`: skip (dead identities cannot be superseded).
   - If `vna_threshold` is not null and `MTP(B_D) > vna_threshold`: skip (identity was expired when this supersession was inscribed).
   - If another supersession from the same source identity has already been applied: skip (only the first supersession per identity is valid).
   - If `D.vnb` is present and `MTP(B) < D.vnb`: record as **pending supersession**; skip for now.
   - Otherwise: set `current_keys = D.k`. Set `state = active`. Set `vna_threshold = D.vna` (or `null` if absent). Cancel any **pending revocations** from the old key set (a scheduled revocation is nullified when the identity supersedes past it — the old keys no longer have authority over the new identity).

4. **Evaluate expiry.** After processing all documents: if `state = active` and `vna_threshold` is not null and `MTP(B) > vna_threshold`: set `state = expired`.

5. **Evaluate pending documents.** The algorithm is stateless — it re-evaluates all documents against the current `MTP(B)` on every invocation. A document that was "pending" at an earlier evaluation point will naturally activate when `MTP(B)` reaches its `vnb` threshold on a subsequent evaluation.

   **Supersession escapes scheduled revocation:** If a supersession activates before a pending revocation's `vnb` is reached, the revocation targets the old key set which has been superseded away. On subsequent evaluations, the revocation's `vnb` may be reached, but it no longer applies — the identity has moved to new keys. Implementations MUST NOT apply a revocation that targets a key set which was superseded before the revocation's `vnb` threshold.

6. **Return** `{ state, current_keys, vna_threshold, genesis_fingerprint }`.

### 6. Constraints on Expired Key Sets

When an identity is `expired`:

- **Attestations** (AIP-05) signed by expired keys MUST be rejected by verifiers.
- **Receipts** (AIP-06) signed by expired keys MUST be rejected.
- **Heartbeats** (AIP-07) signed by expired keys MUST be rejected.
- **Attestation revocations** (AIP-05) signed by expired keys MUST be rejected.
- **Supersession** (AIP-02) signed by expired keys MUST be rejected. Expiry is final — the identity cannot rotate to new keys.
- **Revocation** (AIP-03) signed by expired keys MUST be rejected. Expired keys lose all authority, including revocation authority.
- **Historical documents** signed before expiry remain valid. Expiry is not retroactive and does not imply compromise.

## Examples

### Example 1: Identity with Expiry (JSON)

```json
{
  "v": "1.0",
  "t": "id",
  "n": "TempBot",
  "k": [
    {
      "t": "ed25519",
      "p": "O2onvM62pC1io6jQKm8Nc2UyFXcd4kOmOsBIoYtZ2ik"
    }
  ],
  "vna": 1767225600,
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

This identity expires on 2026-01-01 00:00:00 UTC (Unix timestamp 1767225600). After that time, the keys cannot sign new documents, but historical documents signed before expiry remain valid.

### Example 2: Scheduled Supersession (JSON)

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
  "reason": "key-rotation",
  "vnb": 1740000000,
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

This supersession is inscribed early but doesn't take effect until Unix timestamp 1740000000. The old identity remains active until then.

### Example 3: Scheduled Revocation (JSON)

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
  "vnb": 1767225600,
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

This revocation is scheduled for 2026-01-01 00:00:00 UTC. The identity remains active until then, at which point the entire chain dies.

## Implementation Considerations

### MTP Calculation

Bitcoin MTP is the median of the last 11 block timestamps. Implementations MUST compute this correctly:

```python
def compute_mtp(block_height, blockchain):
    # Collect last 11 blocks (current + 10 prior)
    blocks = [blockchain.get_block(block_height - i) for i in range(11)]
    timestamps = [block.time for block in blocks]
    return sorted(timestamps)[5]  # median of 11 values
```

For blocks near genesis (height < 10), use all available blocks.

### Pending Document Tracking

Explorers SHOULD track pending supersessions and revocations separately from active ones. When displaying identity state, indicate if there are pending lifecycle events and when they activate.

UI example:
```
Identity: Shrike
Status: Active
Scheduled: Supersession on 2026-02-20 (block ~830000)
```

### Cost Considerations

Adding `vna` or `vnb` adds ~20 bytes to a document (integer field + overhead). Minimal cost increase.

## Security Considerations

### Expiry as Poison Pill Mitigation

Expiry limits the poison pill attack surface (AIP-03). An attacker who cracks a historical key can only trigger revocation if:
1. The key was part of a non-expired identity key set, AND
2. The revocation is inscribed before the key set expires

Setting short `vna` lifespans (e.g., 1 year per key set, rotating via supersession) progressively shrinks the revocation attack surface.

Trade-off: More frequent supersessions cost more and create longer chains. Balance security against operational cost.

### Scheduled Supersessions for Safe Rotation

Scheduled supersessions (`vnb` on supersession documents) allow agents to pre-commit to key rotation at a future time. This is useful for:
- **Coordinated migrations** — Multiple agents rotate keys simultaneously
- **Policy compliance** — Enforcing regular key rotation schedules
- **Grace periods** — Announcing rotation in advance so verifiers can prepare

If an attacker compromises the old key before the scheduled supersession activates, they can create a competing immediate supersession. The first to confirm wins (AIP-02 §3). Scheduled supersessions do NOT defend against active compromises — only against passive adversaries who wait.

### Scheduled Revocations for Lifecycle Management

Scheduled revocations (`vnb` on revocation documents) are useful for:
- **Planned shutdowns** — Agent announces end-of-life in advance
- **Dead man's switch** — If the agent doesn't cancel the scheduled revocation (via supersession), the identity auto-destructs

A scheduled revocation can be nullified by superseding before the `vnb` threshold is reached (§5, step 3b).

## TypeScript Interface

```typescript
/** Validity windows are integers (Unix seconds) */
type UnixTimestamp = number;

/** Identity document with optional expiry */
interface IdentityDocument {
  // ... (other fields from AIP-01)
  /** Valid-not-after: identity expiry (Unix seconds, optional) */
  vna?: UnixTimestamp;
}

/** Supersession document with optional scheduled rollover and expiry */
interface SupersessionDocument {
  // ... (other fields from AIP-02)
  /** Valid-not-before: scheduled rollover (Unix seconds, optional) */
  vnb?: UnixTimestamp;
  /** Valid-not-after: expiry for the resulting key set (Unix seconds, optional) */
  vna?: UnixTimestamp;
}

/** Revocation document with optional scheduled activation */
interface RevocationDocument {
  // ... (other fields from AIP-03)
  /** Valid-not-before: scheduled revocation (Unix seconds, optional) */
  vnb?: UnixTimestamp;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)
- [AIP-03: Revocation](./aip-03.md)
- [Bitcoin MTP (BIP 113)](https://github.com/bitcoin/bips/blob/master/bip-0113.mediawiki)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-04 defines validity windows: expiry, scheduled activation, and MTP-based time evaluation.*

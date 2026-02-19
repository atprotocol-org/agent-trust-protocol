# AIP-07: Heartbeats

**Status:** Draft  
**Author(s):** ATP Core Contributors  
**Created:** 2026-02-16  
**Updated:** 2026-02-16  
**Dependencies:** AIP-01

## Abstract

AIP-07 defines **heartbeats** — lightweight signed documents proving an identity holder is active at a point in time. Heartbeats provide liveness signals without the weight of full identity operations. They include a monotonically increasing sequence number to prevent replay attacks. Heartbeats can be inscribed on-chain or published through any medium — the signature provides proof of liveness regardless of delivery mechanism.

## Motivation

Agents need to prove liveness:
- **Activity monitoring** — Distinguish active agents from abandoned identities
- **Service availability** — Signal that an agent is operational and reachable
- **Dead man's switch** — Absence of heartbeats can trigger alerts or automated actions
- **Reputation factor** — Consistent heartbeat history indicates reliable, long-running operation

Heartbeats are intentionally lightweight (~180 bytes JSON, ~110 bytes CBOR) to minimize inscription costs for regular liveness proofs.

## Specification

### 1. Heartbeat Document Structure

A lightweight signed document proving liveness.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `v` | string | `"1.0"` | Version |
| `t` | string | `"hb"` | Document type |
| `f` | binary | — | Identity fingerprint (`k[0]` fingerprint) |
| `ref` | location-ref | — | Location reference to signer's identity |
| `seq` | integer | monotonically increasing from 0 | Sequence number |
| `s` | signature | `{ f, sig }` | Signature |
| `ts?` | integer | Unix seconds | Created timestamp (informational) |
| `msg?` | string | — | Status message (optional) |

**Note:** The top-level `f` is the identity fingerprint (`k[0]` fingerprint), while `s.f` is the fingerprint of the specific key that signed the heartbeat. These are the same when the primary key signs, but differ when a secondary key signs.

#### 1.1 Sequence Number (`seq`)

The `seq` field prevents replay attacks. Each heartbeat MUST have a `seq` value strictly greater than any previously seen heartbeat from the same identity fingerprint.

Verifiers MUST reject heartbeats with a `seq` equal to or less than the highest previously observed value. Since `seq` is included in the signed payload, replaying an old heartbeat with an updated `seq` is cryptographically impossible without the private key.

Sequence numbers start at 0 for the first heartbeat and increment by 1 for each subsequent heartbeat. Gaps are allowed (an agent can skip sequence numbers), but strict monotonic increase is required.

In case of chain reorganizations, verifiers MAY reassess heartbeat sequence validity based on the canonical chain. Heartbeats on orphaned blocks SHOULD be discarded.

#### 1.2 Status Message (`msg`)

The optional `msg` field provides a human-readable status update. Common uses:
- Current activity: `"Processing research backlog"`
- Service announcement: `"API endpoint migrated to new server"`
- Availability window: `"Active until 2026-03-01"`
- Version info: `"Running ATP v1.1"`

Keep it concise — heartbeats are for liveness signals, not long-form communication (use publications for that — see AIP-08).

### 2. Heartbeat Frequency

Heartbeats do not affect identity validity — an identity without heartbeats is not invalid. They provide an optional activity signal for agents and explorers that track liveness.

Recommended frequency:
- **High-value services**: Daily heartbeats
- **Standard agents**: Weekly heartbeats
- **Low-activity agents**: Monthly heartbeats

Agents SHOULD publish heartbeats at a consistent cadence. Large gaps in heartbeat history signal potential abandonment or operational issues.

### 3. Off-Chain Heartbeats

Heartbeats CAN be published off-chain (e.g., via HTTP API, relay network, peer-to-peer messaging) rather than inscribed on Bitcoin. The signature provides proof of liveness regardless of delivery mechanism.

Off-chain heartbeats:
- **Pros**: No inscription cost, higher frequency possible
- **Cons**: Not permanently archived, requires trust in the relay/API

For critical liveness proofs (e.g., demonstrating continuous operation over years), on-chain heartbeats provide stronger evidence. For routine activity signals, off-chain heartbeats are sufficient.

Explorers MAY accept off-chain heartbeats via API submission and cache them for display, but SHOULD clearly distinguish between on-chain (permanent) and off-chain (ephemeral) heartbeats.

## Examples

### Example: Basic Heartbeat (JSON)

```json
{
  "v": "1.0",
  "t": "hb",
  "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
  "ref": {
    "net": "bip122:000000000019d6689c085ae165831e93",
    "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
  },
  "seq": 42,
  "ts": 1738627200,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

### Example: Heartbeat with Status Message (JSON)

```json
{
  "v": "1.0",
  "t": "hb",
  "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
  "ref": {
    "net": "bip122:000000000019d6689c085ae165831e93",
    "id": "6ffcca0cc29da514e784b27155e68c3d4c1ca2deeb6dc9ce020a4d7e184eaa1c"
  },
  "seq": 43,
  "msg": "Migrating to new infrastructure — expect brief downtime",
  "ts": 1738713600,
  "s": {
    "f": "xK3jL9mN1qQ9pE4tU6u1fGRjwNWwtnQd4fG4eISeI6s",
    "sig": "<86 base64url characters>"
  }
}
```

## Verification

### Heartbeat Verification Procedure

1. Decode document
2. Verify `t` is `"hb"`
3. Resolve `ref` to fetch the identity document (may be `t: "id"` or `t: "super"`)
4. Verify the top-level `f` matches the identity's fingerprint (computed from identity's `k[0]`)
5. Find a key in the identity's key set whose fingerprint matches `s.f`. Reject if no match.
6. Remove `s` field, re-encode in canonical form (AIP-01 §5.2)
7. Prepend domain separator `ATP-v1.0:` and verify `s.sig` using the matched key (AIP-01 §4.2, §4.4)
8. Verify `seq` is strictly greater than the highest previously observed `seq` for this identity fingerprint. Reject if equal or lower.
9. If `ts` is present, verify it is reasonable (not more than 2 hours from current time)

## Implementation Considerations

### Sequence Number Tracking

Explorers MUST track the highest observed `seq` per identity fingerprint. When a new heartbeat arrives:

```python
def verify_heartbeat_sequence(identity_fingerprint, new_seq):
    highest_seq = db.get_highest_seq(identity_fingerprint) or -1
    if new_seq <= highest_seq:
        return False  # Reject: sequence not strictly increasing
    db.update_highest_seq(identity_fingerprint, new_seq)
    return True
```

For agents that rotate identities via supersession (AIP-02), sequence tracking is per identity fingerprint, not per genesis fingerprint. When an identity supersedes, the new identity starts its sequence from 0.

Alternatively, agents MAY maintain a single sequence counter across supersessions by coordinating sequence numbers across the chain. This is more complex but provides a unified liveness history.

### Heartbeat Indexing

Explorers SHOULD index heartbeats:
- **By identity fingerprint** — Latest heartbeat per identity
- **By sequence** — Detect gaps in heartbeat history
- **By timestamp** — Identify stale identities (no heartbeat in X days/weeks)

Provide APIs like:
- `/identity/<fingerprint>/heartbeats` — Full heartbeat history
- `/identity/<fingerprint>/latest-heartbeat` — Most recent heartbeat
- `/active-identities?since=<timestamp>` — Identities with heartbeats after timestamp

### Cost Considerations

| Document | JSON Size | CBOR Size | Est. Cost (USD) |
|----------|-----------|-----------|-----------------|
| Heartbeat | ~180 bytes | ~110 bytes | $0.50-1.50 |

For agents that publish daily heartbeats on-chain, annual cost is roughly $180-550 at current fee rates. For most use cases, off-chain heartbeats (submitted via explorer API) are more economical.

## Security Considerations

### Replay Attack Prevention

The `seq` field combined with signature over the full document prevents replay attacks. An attacker who intercepts an old heartbeat cannot:
- Replay it as-is (verifiers reject `seq` ≤ highest seen)
- Modify `seq` (signature verification fails)

### Liveness != Trustworthiness

Heartbeats prove an identity is **active**, not that it's **trustworthy**. A malicious agent can publish heartbeats regularly. Explorers SHOULD NOT use heartbeat presence as the sole trust signal.

Combine liveness with other factors:
- Attestations (AIP-05)
- Receipt history (AIP-06)
- Identity age and inscription costs (AIP-01)
- Supersession history (AIP-02)

### Heartbeat Gaps and Dead Man's Switches

Absence of heartbeats can indicate:
- **Normal inactivity** — Agent is offline but identity is still controlled
- **Abandonment** — Agent has stopped operating
- **Key compromise** — Attacker controls identity but doesn't know about heartbeat expectations

Explorers SHOULD:
- Flag identities with large heartbeat gaps (e.g., no heartbeat in 30+ days)
- Allow users to set alerts for heartbeat absence
- NOT automatically revoke or invalidate identities based on heartbeat gaps

Third-party services MAY implement dead man's switches using heartbeat absence as a trigger, but this is outside the scope of ATP itself.

## TypeScript Interface

```typescript
/** Heartbeat document */
interface HeartbeatDocument {
  v: "1.0";
  t: "hb";
  /** Identity fingerprint (k[0] fingerprint) */
  f: Uint8Array;
  /** Location of the signer's identity document */
  ref: LocationRef;
  /** Sequence number (monotonically increasing from 0) */
  seq: number;
  s: Signature;
  ts?: number;
  /** Optional status message */
  msg?: string;
}
```

## References

- [AIP-01: Identity Documents & Signing](./aip-01.md)
- [AIP-02: Supersession](./aip-02.md)

## Changelog

### 1.0 (2026-02-16)

- Initial specification extracted from ATP v1.0 monolithic spec
- Status: Draft

---

*AIP-07 defines heartbeats: lightweight liveness proofs with replay protection.*

# Storage Bump Strategy

## Overview

Soroban uses a rent model where ledger entries (persistent storage) have a TTL
(time-to-live) measured in ledgers. If an entry is not accessed or bumped within
its TTL, it may be archived and become unavailable. OphirPay implements a
two-tier bump strategy to ensure long-lived entries stay live.

## TTL Parameters

| Parameter | Value | Meaning |
|---|---|---|
| `min_ttl` | 5,000 ledgers | Minimum TTL to maintain (~8.5 hours at 5s/ledger) |
| `max_ttl` | 50,000 ledgers | Maximum TTL to extend to (~3.5 days at 5s/ledger) |

These values are chosen to:
- **Avoid archival**: 50,000 ledgers at ~5 seconds each = ~2.9 days of headroom
- **Minimize rent cost**: Only extend to 50,000, not to the protocol maximum
- **Balance maintenance frequency**: A bump every ~3.5 days is manageable

## Storage Tiers

### Instance Storage (contract-level)

Instance storage holds counters, configuration, and ownership data. It is
bumped on every write via `env.storage().instance().extend_ttl(5000, 50000)`.

**Entries**: `PAYMENT_COUNT`, `ESCROW_COUNT`, `STREAM_COUNT`, `BATCH_COUNT`,
`OWNER`, `PAUSED`, `VERSION`, `MULTISIG_CONFIG`, `FEE_KEY`, `FEE_COLL`,
`GOV_CONF`, `EMITTER_ADDR`, `REENTRANCY_LOCK`, `LOCKED_BALANCE`, and all
stat counters.

### Persistent Storage (entry-level)

Persistent storage holds individual records: payments, escrows, streams,
batches, audit entries, refunds, proposals, hooks, votes, spending limits,
roles, timelocked actions, and configuration versions.

**Bump on write**: Every persistent write is immediately followed by
`extend_ttl(5000, 50000)` to keep the entry live.

### Persistent entries without regular writes

Some persistent entries may not be written to for extended periods:
- **Audit entries**: Written once, never modified
- **Vote records**: Written once per proposal per voter
- **Old payment/escrow/stream records**: May not be read for months

These entries rely on the maintenance function to extend their TTL.

## Maintenance Function

The `maintain_storage_bump` function allows the contract owner to extend
the TTL of old persistent entries. It accepts a range of entry IDs and
bumps them if they exist.

### Gas Budget

Each `extend_ttl` call on a persistent entry costs approximately:
- **Read**: ~200 gas (load the entry metadata)
- **Write**: ~200 gas (update the TTL)
- **Total per entry**: ~400 gas

For a batch of 50 entries: ~20,000 gas (~0.02 XLM at 100,000 stroops/unit)

### Recommended Maintenance Schedule

| Entry Type | Frequency | Batch Size | Rationale |
|---|---|---|---|
| Audit entries | Weekly | 50 | Written once, high volume |
| Old payments | Weekly | 50 | Accessed rarely after completion |
| Escrows | Daily (active) | 20 | Active escrows need frequent bumps |
| Streams | Daily (active) | 20 | Active streams need frequent bumps |
| Votes | After voting period | 50 | No longer accessed after execution |
| Proposals | After execution | 50 | No longer modified after execution |

## Write Path Bumps

Every state-changing function that writes to persistent storage immediately
bumps the entry. This ensures:

1. **Hot entries** (recently created/modified) are always live
2. **No archival risk** for entries that are actively used
3. **Predictable cost** — bump gas is accounted for in the function's budget

### Functions with Write-Path Bumps

| Function | Entry Type | Bumped |
|---|---|---|
| `record_payment` | `(PAYMENT_KEY, id)` | ✅ |
| `cancel_payment` | `(PAYMENT_KEY, id)` | ✅ |
| `create_escrow` | `(ESCROW_KEY, id)` | ✅ |
| `release_escrow` | `(ESCROW_KEY, id)` | ✅ |
| `claim_escrow` | `(ESCROW_KEY, id)` | ✅ |
| `create_stream` | `(STREAM_KEY, id)` | ✅ |
| `claim_stream` | `(STREAM_KEY, id)` | ✅ |
| `cancel_stream` | `(STREAM_KEY, id)` | ✅ |
| `create_batch` | `(BATCH_KEY, id)` | ✅ |
| `record_audit` | `(AUDIT_LOG_KEY, id)` | ✅ |
| `request_refund` | `(REFUND_KEY, id)` | ✅ |
| `approve_refund` | `(REFUND_KEY, id)` | ✅ |
| `reject_refund` | `(REFUND_KEY, id)` | ✅ |
| `process_refund` | `(REFUND_KEY, id)` | ✅ |
| `propose_payment` | `(APPROVAL_KEY, id)` | ✅ |
| `approve_payment` | `(APPROVAL_KEY, id)` | ✅ |
| `execute_approved_payment` | `(APPROVAL_KEY, id)` | ✅ |
| `create_proposal` | `(PROPOSAL_KEY, id)` | ✅ |
| `vote_on_proposal` | `(VOTE_KEY, proposal, voter)` | ✅ |
| `execute_proposal` | `(PROPOSAL_KEY, id)` | ✅ |
| `set_spending_limit` | `(SPEND_LIMIT_KEY, user)` | ✅ |
| `atomic_spend` | `(SPEND_LIMIT_KEY, user)` | ✅ |
| `grant_role` | `(ROLE_KEY, grantee)` | ✅ |
| `register_hook` | `(HOOK_KEY, id)` | ✅ |
| `unregister_hook` | `(HOOK_KEY, id)` | ✅ |
| `propose_timelocked_action` | `(TIMELOCK_KEY, id)` | ✅ |
| `execute_timelocked_action` | `(TIMELOCK_KEY, id)` | ✅ |
| `cancel_timelocked_action` | `(TIMELOCK_KEY, id)` | ✅ |
| `set_multisig_config` | `(MSIG_VER_CNT, ver)` | ✅ |
| `set_fee_config` | `(FEE_VER_CNT, ver)` | ✅ |

## Gas Cost Accounting

The bump gas cost is transparent in the contract's gas budget:

- **Per-write bump**: ~400 gas (included in function cost)
- **Maintenance bump**: ~400 gas × batch size
- **No surprise archival**: Entries are always live when accessed

## Monitoring

The contract exposes `get_storage_info` to check:
- Current instance storage TTL
- Whether persistent entries need bumping

Operators should monitor:
- TTL of oldest persistent entries
- Frequency of maintenance calls
- Gas consumption of maintenance transactions

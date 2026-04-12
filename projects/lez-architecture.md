---
title: "LEZ Architecture: L1 State Access"
tags: [logos, lez, svm, architecture]
source: logos-blockchain/logos-execution-zone (GitHub)
reviewed: false
---

# LEZ Architecture: Can the SVM Read Logos L1 State?

## TL;DR

**No.** The LEZ SVM cannot read the Logos L1 epoch nonce or block header data. The LEZ and L1 operate in separate execution contexts with no direct state cross-read mechanism. An oracle integration (or L1-native randomness contract) is required to bring L1 consensus randomness into LEZ.

---

## Architecture Overview

The Logos stack has three functional layers:

| Layer | Component | Role |
|-------|-----------|------|
| L1 | Logos/Bedrock (Cryptarchia PPoS) | Consensus, data availability, epoch randomness |
| Bridge | Indexer | Parses L2 block inscriptions from L1; tracks L1 block IDs |
| L2 | LEZ Sequencer + SVM | Executes LEZ programs; produces L2 blocks |

Source: `logos-blockchain/logos-execution-zone` [GitHub](https://github.com/logos-blockchain/logos-execution-zone).

---

## How L2 Blocks Reach L1

When the LEZ sequencer produces a block:

1. The L2 block (`HashableBlockData`) is serialised via `borsh::to_vec()`.
2. The serialised bytes are inscribed as an `Op::ChannelInscribe` operation in a Mantle transaction.
3. This transaction is submitted to L1 via the `BlockSettlementClient`.
4. L1 validators confirm the transaction; the L2 block data becomes part of the L1 ledger.

Source: `sequencer/core/src/lib.rs` (`produce_new_block_with_mempool_transactions`) and `sequencer/core/src/block_settlement_client.rs`.

**Key implication:** L2 block data is encoded inside L1 transaction payloads. It is not stored in or accessible from L1 block headers.

---

## What the Indexer Tracks

The indexer (`indexer/core/src/lib.rs`) maintains the relationship between L2 blocks and their hosting L1 block:

```rust
pub struct BackfillBlockData {
    l2_blocks: Vec<Block>,   // L2 blocks parsed from the L1 tx inscription
    l1_header: HeaderId,     // L1 block header ID that confirmed this L2 data
}
```

The indexer stores `l1_header` (the L1 `HeaderId`) in its local RocksDB, but this is an internal bookkeeping detail. The indexer RPC (`subscribe_to_finalized_blocks`) only exposes the L2 `BlockId`:

```rust
// indexer/service/rpc/src/lib.rs
async fn subscribe_to_finalized_blocks(&self) -> SubscriptionResult;
// Returns: BlockId (L2 block ID only)
```

Source: `indexer/core/src/lib.rs`, `indexer/service/src/service.rs`.

**The L1 header ID is never exposed to the sequencer or LEZ programs via any RPC endpoint.**

---

## What LEZ Programs Can Access

LEZ programs execute in the SVM context with only the following state:

- **L2 block height** (`block_id: u64`) from `HashableBlockData`
- **Wall-clock timestamp** (`timestamp: u64`) from `chrono::Utc::now().timestamp_millis()` — not derived from L1 slot or epoch
- **Clock program accounts** (`ClockAccountData { block_id, timestamp }`) — same L2-only data

Source: `programs/clock/core/src/lib.rs`, `sequencer/core/src/lib.rs` (`produce_new_block_with_mempool_transactions`).

The sequencer has no mechanism to call `consensus_info()` and extract the L1 epoch nonce for use in L2 execution. The `BedrockClient` in the block settlement path can call `get_consensus_info()` (returning `CryptarchiaInfo`), but this is used only for transaction submission, not for supplying data to L2 programs.

---

## The CryptarchiaInfo Gap

The `CryptarchiaInfo` struct (defined in `logos-co/nomos/services/chain/chain-service/src/lib.rs`) contains:

```rust
pub struct CryptarchiaInfo {
    pub lib: HeaderId,      // Last irreversible block
    pub lib_slot: Slot,
    pub tip: HeaderId,
    pub slot: Slot,
    pub height: u64,
    pub mode: State,
}
```

**It does not include the epoch nonce (`eta_ep`).** The epoch nonce is computed inside the cryptarchia engine and used for leader election, but it is not exposed via any API endpoint.

Source: `logos-co/nomos/services/chain/chain-service/src/lib.rs` (lines 160–166).

---

## Blockhash Comparison: Solana vs Logos LEZ

For context, Solana contracts can read the blockhash from the current slot leader via the `sysvar` account. This is the source of Solana's blockhash-based randomness surface.

Logos LEZ has **no equivalent**. There is no SVM sysvar or system call that exposes:
- The L1 block header hash
- The L1 slot number
- The L1 epoch nonce

The only cross-layer signal is the L2 block itself (confirmed L2 height and wall-clock time), neither of which is suitable as a randomness source.

---

## Consequences for Randomness

If LEZ applications require onchain randomness that is anchored to the L1 consensus ( Cryptarchia epoch nonce), there are two viable paths:

### Option A: Oracle-Based

Integrate a randomness oracle (e.g. Chainlink VRF or Pyth Entropy) that samples the L1 epoch nonce off-chain and delivers a VRF proof on-chain. This introduces a trust assumption (oracle liveness and honesty) but is immediately available.

See: [[chainlink-vrf]], [[pyth-entropy]].

### Option B: L1 Randomness Contract (Future)

Deploy a randomness contract on Logos L1 that exposes the epoch nonce via a read call. The LEZ would need a mechanism to perform cross-chain reads or receive a callback from L1 at epoch boundaries. This is a protocol-level addition requiring changes to the LEZ/L1 bridge design.

Status: `[DATA NEEDED]` — not currently implemented.

---

## Verdict

| Question | Answer |
|----------|--------|
| Can LEZ SVM read L1 epoch nonce? | **No** |
| Is L1 epoch nonce exposed via any API? | **No** |
| Can the indexer provide L1 header data to LEZ? | **Not currently** — indexer RPC only returns L2 block IDs |
| Is there a sysvar or system call for L1 state? | **No** |
| Can LEZ use L1 consensus randomness without an oracle? | **No** |

## References

- Source repo: [logos-blockchain/logos-execution-zone](https://github.com/logos-blockchain/logos-execution-zone)
- Nomos consensus: [logos-co/nomos](https://github.com/logos-co/nomos)
- LEZ wallet quickstart: [logos-docs/docs/apps/wallet/journeys/quickstart-for-the-logos-execution-zone-wallet.md](https://github.com/logos-co/logos-docs/blob/main/docs/apps/wallet/journeys/quickstart-for-the-logos-execution-zone-wallet.md)
- Block settlement: `sequencer/core/src/block_settlement_client.rs`
- Indexer block parsing: `indexer/core/src/lib.rs` (`parse_block_owned`)

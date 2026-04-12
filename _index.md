---
tags: [index, randomness, ethereum, solana]
---

# Onchain Randomness: Project Index

Overview of onchain randomness solutions researched for the Logos LEZ RFP.

## Landscape Overview

Onchain randomness is a hard problem: blockchains are deterministic, so no native source of entropy exists. Every solution involves either:
1. **Off-chain oracle injection** (Chainlink VRF, Pyth Entropy, Switchboard VRF)
2. **Consensus-layer commit-reveal** (Ethereum RANDAO)
3. **VDF-based delay** (planned for Ethereum, used by Solrand-style approaches)

The fundamental tension: randomness must be unpredictable *before* use, but verifiable *after* use. These solutions each trade off latency, cost, collusion-resistance, and centralisation.

## Project Rankings

| # | Project | Ecosystem | Mechanism | Latency | Trust Model |
|---|---|---|---|---|---|
| 1 | [[chainlink-vrf]] | Ethereum + 20 chains | VRF (oracle) | 2-3 blocks (~24-36s) | Bonded oracle set |
| 2 | [[pyth-entropy]] | Solana, EVM, 40+ chains | Commit-reveal (oracle) | ~2 blocks | Single provider (but blockhash-mixed) |
| 3 | [[switchboard-vrf]] | Solana, 10+ chains | VRF (oracle) | 2-4 blocks | Decentralised oracle queue |
| 4 | [[orao-solana-vrf]] | Solana | VRF (oracle) | ~2 blocks | Oracle network |
| 5 | [[ethereum-randao]] | Ethereum consensus | Commit-reveal (consensus) | 1 epoch (~6.4 min) | Validator set (BLS-signed) |
| 6 | [[ethereum-vdf]] | Ethereum (planned) | RANDAO + VDF (consensus) | ~2 epochs | Validator set + VDF hardware |

## Key Observations

### Ethereum (L1)
- **No native smart-contract-accessible randomness**. Contracts must use an oracle (Chainlink VRF dominates) or commit-reveal schemes
- **Consensus randomness (RANDAO)** exists but is not accessible to EVM contracts; it is used for validator selection only
- **VDFs were planned** but not deployed due to hardware complexity; RANDAO + BLS is the current approach

### Solana
- **Leader controllability of blockhash** is the core problem. A Solana leader can influence the blockhash, making direct blockhash-based randomness insecure
- **ORAO VRF, Switchboard VRF**: Oracle-based VRF solutions that mix blockhash with oracle entropy to reduce manipulability
- **Pyth Entropy**: Uses a pre-committed hash-chain; provider commits many values upfront, reveals one per request mixed with blockhash. Lower cost than full VRF but requires trusting the provider to pre-commit honestly

### Cross-Chain
- Chainlink VRF is the dominant solution across EVM chains by number of integrated contracts (2,200+)
- Pyth Entropy is expanding from Solana to EVM and 40+ chains, leveraging Pyth's existing price feed publisher network
- Switchboard is multi-chain but Solana-native; secures ~$6-6.5B TVL across 50+ projects

## Logos LEZ Applicability

### Critical Finding: LEZ Cannot Read L1 State

After auditing the `logos-blockchain/logos-execution-zone` codebase (the LEZ sequencer and indexer), the following architecture was confirmed:

- **L1 (Logos/Bedrock)** runs Cryptarchia PPoS consensus. The epoch nonce (`eta_ep`) is computed inside the cryptarchia engine and used for leader election, but it is **not exposed via any API endpoint**.
- **The Indexer** parses L2 block inscriptions from L1 and tracks the L1 `HeaderId` internally, but the `subscribe_to_finalized_blocks()` RPC only exposes the L2 `BlockId`, not the L1 header.
- **The LEZ Sequencer** produces L2 blocks using wall-clock timestamps, not any L1-derived value. There is no sysvar, system call, or oracle mechanism that exposes the L1 epoch nonce to the SVM context.
- **`CryptarchiaInfo`** (the consensus info struct served by the L1 node API) contains `lib`, `tip`, `slot`, `height`, and `mode`, but **does not include the epoch nonce**.

**Conclusion:** LEZ SVM programs cannot access Logos L1 consensus randomness without either (a) an oracle, or (b) a protocol-level change to expose the L1 epoch nonce.

Source: [[lez-architecture]]

### Recommended Paths for LEZ Randomness

1. **Oracle-based (available now):** Integrate Chainlink VRF or Pyth Entropy on LEZ. Logos would need to deploy a randomness consumer contract and integrate with the oracle's existing networks.
2. **L1 randomness contract (future):** Deploy a contract on Logos L1 that exposes the epoch nonce at epoch boundaries. LEZ sequencer or a bridge module would call this contract and relay the value to LEZ programs. Requires protocol-level addition.
3. **LEZ-native commit-reveal:** Build a commit-reveal scheme directly into the LEZ sequencer, independent of L1 consensus randomness.

### No Direct L1 Read Possible

Unlike Solana (where contracts can read the blockhash via a sysvar account), LEZ has no equivalent mechanism. The L2 block is inscribed to L1 as transaction data, but L1 block headers (containing consensus metadata) are not readable from the SVM execution context.

## Pending Fact-Checking

All 6 PRs below are open and awaiting fact-checker review:
- PR #1: [[https://github.com/marclawclaw/research-onchain-randomness/pull/1 | Chainlink VRF]]
- PR #2: [[https://github.com/marclawclaw/research-onchain-randomness/pull/2 | Switchboard VRF]]
- PR #3: [[https://github.com/marclawclaw/research-onchain-randomness/pull/3 | Pyth Entropy]]
- PR #4: [[https://github.com/marclawclaw/research-onchain-randomness/pull/4 | Ethereum RANDAO]]
- PR #5: [[https://github.com/marclawclaw/research-onchain-randomness/pull/5 | Ethereum VDF Randomness]]
- PR #6: [[https://github.com/marclawclaw/research-onchain-randomness/pull/6 | ORAO Solana VRF]]
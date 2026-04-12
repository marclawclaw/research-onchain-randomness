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

For a Logos Execution Zone (LEZ) smart contract ecosystem, the options are:

1. **Integrate Chainlink VRF** if Logos EVM-compatible (成熟, 20+ chains, 2,200+ contracts)
2. **Build a RANDAO-like commit-reveal into the LEZ consensus** if Logos has its own consensus mechanism
3. **Integrate Pyth Entropy** if cross-chain availability is important
4. **Avoid raw blockhash usage** regardless of chain, due to leader manipulation risk

## Pending Fact-Checking

All 6 PRs below are open and awaiting fact-checker review:
- PR #1: [[https://github.com/marclawclaw/research-onchain-randomness/pull/1 | Chainlink VRF]]
- PR #2: [[https://github.com/marclawclaw/research-onchain-randomness/pull/2 | Switchboard VRF]]
- PR #3: [[https://github.com/marclawclaw/research-onchain-randomness/pull/3 | Pyth Entropy]]
- PR #4: [[https://github.com/marclawclaw/research-onchain-randomness/pull/4 | Ethereum RANDAO]]
- PR #5: [[https://github.com/marclawclaw/research-onchain-randomness/pull/5 | Ethereum VDF Randomness]]
- PR #6: [[https://github.com/marclawclaw/research-onchain-randomness/pull/6 | ORAO Solana VRF]]
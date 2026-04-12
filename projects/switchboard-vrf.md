---
tags: [project, solana, oracle, vrf, randomness]
source: https://switchboardxyz.medium.com/verifiable-randomness-on-solana-46f72a46d9cf
---

# Switchboard VRF

Switchboard VRF is the first Verifiable Random Function (VRF) implementation on Solana, enabling Solana programs to request provably random values from a decentralised oracle network.

## Overview

Switchboard is a decentralised oracle network live on 10+ chains and securing an estimated $6.5 billion in TVL across 51+ projects, including Kamino, Jito, MarginFi, and Drift (Bitget, 2024). In May 2024, Switchboard raised $7.5 million in a Series A funding round (Bitget, 2024).

Switchboard is the only provider offering verifiable randomness on Solana as of July 2022 (Switchboard Medium, 2022).

## Technical Design

### VRF Request Flow

Developers create a VRF account and submit a randomness request. Off-chain oracles compute a VRF signature using a randomly seeded counter combined with the recent blockhash, then post the result back on-chain. The VRF counter is incremented on every randomness request to ensure uniqueness (Switchboard Medium, 2022).

### Security Model

Switchboard's VRF implementation adheres to IRTF CFRG draft volume 11 for VRF (Switchboard Medium, 2022). After an oracle posts the proof on-chain, anyone can verify it. Malicious oracles that submit a false proof will have it rejected on-chain during verification.

The VRF input combines the VRF counter and the recent blockhash to prevent oracles from gaining an unfair advantage. However, a collusion vector between the oracle and the block-producer still exists; Switchboard acknowledges this but notes it is much more secure than using the blockhash alone for randomness (Switchboard Medium, 2022).

### Reliability Improvements

Switchboard oracles migrated to using nonce accounts or non-expiring transactions to submit VRF proofs. This reduces the oracle memory footprint and increases the overall reliability of successfully completing VRF proofs (Switchboard Medium, 2022).

### Cost Structure

At launch, Switchboard set the VRF request cost at 0.1 SOL per request to protect oracles from being overwhelmed (Switchboard Medium, 2022). A July 2022 update reduced the cost to just under 0.002 SOL, a 50x reduction (Switchboard Medium, 2022; X/@switchboardxyz, 2022).

With storage and the PRE v0.4.0 upgrade, the cost is approximately 0.07 SOL per request, of which most is gas and Solana storage rent. The v0.4.0 upgrade automatically returns 0.04 SOL (Solana StackExchange, 2024).

### Developer Experience

Switchboard VRF is Anchor-compatible and supports Cross-Program Invocations (CPI), making it straightforward to integrate into Anchor-based Solana programs (GuidoDipietro GitHub, 2022).

The [Anchor VRF Example Program](https://github.com/switchboard-xyz/switchboard-v2/tree/main/programs/anchor-vrf-parser) provides a reference implementation.

## VRF Account Pool Schema

Third-party implementations such as the [solana-switchboard-vrf-pool](https://github.com/GuidoDipietro/solana-switchboard-vrf-pool) demonstrate an alternative schema using a pool of VRF accounts to allow multiple simultaneous users. Each VRF account acts as a dedicated queue position, and users share a pool of accounts to balance cost and throughput.

## Comparison tonaive Blockhash Randomness

Using the latest blockhash for randomness is low-cost but extremely insecure. Slot leaders can exploit games or protocols by adding noise to blocks they produce to influence the random output. With MEV infrastructure such as Flashbots, Jito Labs, and proposer-builder separation, this attack vector is increasingly accessible (Switchboard Medium, 2022).

EdDSA-based VRFs are also insecure because EdDSA signatures are non-deterministic, meaning the same input can produce different valid signatures (Switchboard Medium, 2022).

Real-world examples where VRF could have prevented exploits include the COPE roulette exploit (which used blockhash for randomness) and the Trash Panda mint sniper (which used predictable mint ordering) (Switchboard Medium, 2022).

## Resources

- Documentation: https://docs.switchboard.xyz/randomness
- Anchor VRF Example: https://github.com/switchboard-xyz/switchboard-v2/tree/main/programs/anchor-vrf-parser
- Rust Crate: https://crates.io/crates/switchboard-solana
- VRF Pool Example: https://github.com/GuidoDipietro/solana-switchboard-vrf-pool
- Discord: https://discord.com/invite/sNeGymrabT

## Tags

#project #solana #oracle #vrf #randomness

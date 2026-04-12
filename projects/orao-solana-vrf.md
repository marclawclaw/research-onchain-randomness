---
tags: [project, solana, oracle, vrf, randomness]
source: https://github.com/orao-network/solana-vrf
---

# ORAO Solana VRF

ORAO (ora.network) is a multi-node Verifiable Random Function oracle network that provides on-chain randomness for Solana programs. The ORAO Solana VRF SDK was one of the earliest VRF implementations on Solana and remains actively maintained.

## Overview

ORAO VRF generates verifiable, unbiased randomness on Solana. The SDK is built using the Anchor framework and is Anchor-compatible. Developers can request randomness via Cross Program Invocation (CPI) calls from their own programs, or through off-chain Rust and JS web3 SDKs.

The repository is at `github.com/orao-network/solana-vrf` and packages are published to npm (`@orao-network/solana-vrf`) and crates.io (`orao-solana-vrf`).

Use cases include: generating unique NFT characteristics, randomising in-game item attributes and weapon levels, algorithmic art generation, and secure verifiable lotteries.

## Technical design

### VRF architecture

ORAO VRF uses a counter and the recent blockhash as inputs to produce its output. This combination is designed to prevent oracle manipulation by tying the randomness to values the leader cannot fully control in isolation.

The SDK supports two program versions:

- Classic VRF: program ID `VRFzZoJdhFWL8rkvu87LpKM3RbcVezpMEc6X5GVDr7y` (devnet and mainnet)
- Callback VRF: program ID `VRFCBePmGTpZ234BhbzNNzmyg39Rgdd6VgdfhHwKypU` (devnet and mainnet)

The v0.4.0 release introduced a new account structure. Developers must use `RandomnessAccountData` to decode fulfilled randomness accounts, as the old deserialisation approach fails on accounts created under v0.4.0 and later.

### CPI integration

ORAO VRF is designed for composability via Solana's Cross Program Invocation. A developer program can call the VRF program within the same transaction, allowing atomic request-and-response patterns. The SDK provides a Rust crate (`orao-solana-vrf`) with a `cpi` feature that exposes the necessary account structs and instruction builders.

The minimum CPI accounts required for a `RequestV2` call are: `payer` (VRF client), `network_state` (VRF on-chain state), `treasury` (fees collected here), `request` (PDA storing the result), and `system_program`.

An Anchor-based Russian Roulette example in the repository demonstrates the full CPI pattern: the player contract requests randomness, waits for fulfilment, then reads the result in a follow-up transaction.

### Fee structure

The v0.4.0 upgrade significantly reduced VRF request costs. According to the AdevarLabs blog, the base VRF fee is 0.001 SOL, with the majority of on-chain cost coming from rent for the request account. Earlier versions (prior to v0.4.0) charged approximately 0.07 SOL per request. The fee change reflects the move to a more efficient account model.

### Performance

Recent performance updates are advertised as achieving sub-second fulfilment. The randomness is not immediately available within the same transaction that requests it; applications must wait for the VRF oracle network to fulfil the request before reading the result.

## Solana blockhash manipulation context

ORAO VRF exists partly because raw Solana blockhashes are unsuitable as randomness sources on their own. This section covers why.

### The blockhash manipulation problem

Unlike Bitcoin, where miners cannot influence the block hash without expending significant energy, a Solana leader node can change the blockhash for little cost. If a Solana program uses the current blockhash as the sole source of randomness, a malicious leader could grind the blockhash to produce a favourable outcome and then withhold or modify blocks accordingly.

A leader could also front-run transactions by selecting which blockhash ends up in the chain, giving them an unfair advantage in any application that depends on randomness.

This vulnerability is not merely theoretical. Candy Machine v2, which used the difference between the last slot hash and clock time to order NFT mint transactions, was exploited through this mechanism.

### The Solrand approach: VDF as a countermeasure

The Solrand project (documented on Devpost) proposed using a Verifiable Delay Function (VDF) on top of the blockhash to close the manipulation window. The approach is:

1. Take the Solana blockhash as input to a VDF.
2. The VDF output becomes the random number.
3. Because the VDF takes approximately 10 seconds or more to compute after the blockhash is known, the leader cannot control the outcome: by the time the random number is calculated, the leader has already passed their block generation window.

Solana uses a SHA-256 hash chain to checkpoint the ledger and coordinate consensus. Solana's implementation is described as VDF-like but is not a true VDF, because verification takes the same amount of time as computation (a true VDF can be verified much faster than it is computed).

A limitation of the VDF approach, noted by AdevarLabs, is that users who know the blockhash in advance can pre-compute the VDF output offline and choose whether to submit their transaction based on the outcome. This means the VDF approach narrows the manipulation window but does not eliminate it entirely. Solrand recommended that applications select randomness from approximately 3 seconds or more in the future to account for propagation delays and narrow the remaining attack surface.

ORAO VRF addresses the blockhash manipulation problem differently: by using a multi-node oracle network with a VRF proof, rather than relying on any single leader's blockhash output.

## Comparison with other Solana VRF protocols

| Protocol | Trust model | Latency | Key mechanism |
|---|---|---|---|
| ORAO VRF | Multi-node oracle, Byzantine Quorum | Sub-second fulfilment | VRF proof from off-chain oracles |
| Switchboard VRF v2 | Oracle network | Multi-round | VRF using counter + blockhash |
| Switchboard SRS (v3) | Intel SGX enclaves | Single transaction | Trusted execution environment |
| Pyth Entropy | Two-party commit-reveal | Two-phase | Hash-chain commit, blockhash mix |

Source for comparison table: AdevarLabs blog on Solana randomness.

## References

- ORAO Solana VRF GitHub repository: https://github.com/orao-network/solana-vrf
- ORAO VRF Rust crate: https://crates.io/crates/orao-solana-vrf
- ORAO VRF npm package: https://www.npmjs.com/package/@orao-network/solana-vrf
- AdevarLabs, "On-Chain Randomness on Solana: Predictability, Manipulation and Safer Alternatives (Part 1)": https://www.adevarlabs.com/blog/on-chain-randomness-on-solana-predictability-manipulation-safer-alternatives-part-1
- Solrand Devpost: https://devpost.com/software/solrand
- Solana Stack Exchange, "How to generate random numbers on-chain": https://solana.stackexchange.com/questions/45/how-to-generate-random-numbers-on-chain [NOT RETRIEVED - 403]

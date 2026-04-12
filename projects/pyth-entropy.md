---
tags: [project, solana, oracle, commit-reveal, randomness, evm]
source: https://www.pyth.network/entropy
---

# Pyth Entropy

Pyth Entropy is the on-chain random number generator (RNG) from the Pyth Network. It delivers secure, verifiable, and low-latency randomness to smart contracts across more than 40 blockchains.

## Overview

Pyth Entropy was launched in March 2024 as Pyth's answer to the lack of fast, cheap, and trustworthy on-chain randomness. The protocol is built on a two-party commit-reveal cryptographic scheme that is secure as long as at least one participant is honest.

The key innovation is that one party (the provider) pre-commits a long hash-chain of random values up front. Users of the protocol then simply consume the next value in the sequence, minimising the number of on-chain transactions required for each randomness request.

Initial support was on EVM chains including Arbitrum, Blast, Chiliz Chain, Mode, LightLink, and Optimism. Pyth Entropy has since expanded to more than 40 blockchains total (source: https://www.pyth.network/entropy).

## Protocol Design

Entropy implements an optimised two-party commit-reveal protocol. The basic commit-reveal flow works as follows:

1. Both parties independently generate a secret random number.
2. Each party hashes their number and commits the hash to the blockchain.
3. Both parties reveal their original numbers.
4. Each party verifies the other's revealed number matches the committed hash.
5. The final random number is computed as `hash(xA, xB)`.

This protocol is secure as long as either party is honest: they draw their value at random and keep it secret until the reveal phase.

Entropy modifies this basic protocol to make it efficient on-chain. Instead of both parties posting on-chain for every request, the provider pre-commits a sequence of N random values using a hash chain:

- `x_(N-1) = random()`
- `x_i = hash(x_(i+1))` for each `i` from `0` to `N-2`

The provider commits to `x_0` on the Entropy contract. Each value can be verified against the previous one by hashing.

To request a random number, a user:

1. Samples their own secret contribution `x_U` and submits it on-chain.
2. The contract assigns an incrementing sequence number representing which provider value the user will receive.
3. After sufficient block confirmations, the provider reveals `x_i` on-chain.
4. The contract verifies `hash(x_i) == x_(i-1)` and computes `r = hash(x_i, x_U)`.
5. The contract delivers `r` via a callback to the calling contract.

The blockhash is mixed into the result to add an additional source of randomness. This mitigates (but does not eliminate) the leader-manipulation issue on Solana, where block leaders could influence outcomes by withholding or inserting blocks.

Source: https://docs.pyth.network/entropy/protocol-design

### Trust Assumptions

The protocol requires several trust assumptions:

- **Provider honesty**: Providers must reveal `x_i` regardless of the final result. Providers can compute `r` off-chain before revealing `x_i`, which permits a censorship attack.
- **No front-running**: Providers who observe user transactions in the mempool can manipulate the result by inserting additional requests or rotating their commitment.
- **Hash-chain secrecy**: Anyone with the hash chain can predict the result of a randomness request before it is requested, and therefore manipulate the outcome. This includes blockchain validators who can use this information to reorder user transactions.

Source: https://docs.pyth.network/entropy/protocol-design

## Entropy v2

Entropy v2 was released with several improvements to the original protocol:

1. **Multiple request variants**: Developers can choose from basic requests, custom gas limit requests, custom provider requests, and full-control requests that specify all parameters.
2. **Enhanced callback status**: Callback statuses allow users to track the status of their callbacks and handle failures.
3. **Entropy Explorer**: A public web interface at entropy-explorer.pyth.network lets teams track and debug callback issues and re-request failed callbacks on-chain.

Entropy v2 maintains backward compatibility with v1. Existing applications can continue using v1, but new applications are encouraged to use v2.

Source: https://docs.pyth.network/entropy/whats-new-entropyv2

## Properties

**Security**: Entropy is built on a commit-reveal protocol. The result is random as long as either the provider or the user is honest. The protocol is trustless in that neither party needs to trust the other beyond those assumptions. The default provider implementation is open source.

**Latency**: Entropy follows a pull design similar to Pythnet Price Feeds. The two parties communicate over HTTP rather than purely on-chain, which reduces latency. Randomness is typically available within a few blocks.

**Ease of use**: Integration requires just a few lines of code. No registration is required. Developers on EVM chains can get started in under 5 minutes using the provided SDK and code examples.

**Cost**: Entropy is designed to be cost-efficient for production use. Users pay a fee in the chain's native token (native gas fees).

**Uptime**: Pyth claims over 99.9% uptime for Entropy services.

Source: https://www.pyth.network/entropy

## Adoption

Early adopters on EVM chains include:

- **FLAP** (Blast): A bundle market for blue-chip NFT mints.
- **Fungible Flip** (Blast): A coin-flip game with no house rake and true 50/50 odds.
- **SlashToken** (Chiliz Chain): A platform for NFT and token tooling for Web3 projects.
- **Drift Protocol** (Solana): A Solana-based perpetual futures DEX.
- **Jupiter** (Solana): A Solana-based DEX aggregator.

Source: https://www.pyth.network/blog/pyth-entropy-random-number-generation-for-blockchain-apps

## Comparison to Other Solana Randomness Solutions

On Solana, native sources of randomness (slot hashes, blockhashes, clock sysvar) are deterministic and can be manipulated by validators. Using these alone is insecure when value is at stake.

Pyth Entropy mitigates this by mixing the blockhash with a provider's committed hash-chain value. This does not eliminate the manipulation risk entirely but raises the bar significantly. By contrast:

- **Switchboard VRF v2**: Uses off-chain oracles to compute a VRF signature (using a randomly-seeded counter plus blockhash) and posts the result with a proof. Replaced by Switchboard Randomness Service (SRS) v3, which uses Intel SGX enclaves for trusted execution in a single transaction.
- **ORAO VRF**: Multi-node VRF oracle with sub-second fulfilment and Byzantine Quorum security. Base VRF fee is 0.001 SOL.
- **Pyth Entropy**: Commit-reveal with hash-chain pre-commitment, mixed with blockhash. Available on 40+ chains including Solana and EVM chains.

Source: https://www.adevarlabs.com/blog/on-chain-randomness-on-solana-predictability-manipulation-safer-alternatives-part-1

## Contract Addresses

Entropy is deployed on multiple EVM and non-EVM chains. Key EVM deployments include:

- Optimism: `0xdF21D137Aadc95588205586636710ca2890538d5`
- Arbitrum: `[DATA NEEDED]`
- Mode: `[DATA NEEDED]`
- Blast: `[DATA NEEDED]`

The default provider address on Optimism is `0x52DeaA1c84233F7bb8C8A45baeDE41091c616506`.

Source: https://www.pyth.network/entropy

## Code

The default provider implementation (Fortuna) is open source and available at:

https://github.com/pyth-network/pyth-crosschain/tree/main/apps/fortuna

The Entropy SDK for EVM developers is at:

https://github.com/pyth-network/pyth-crosschain/tree/main/target_chains/ethereum/entropy_sdk/solidity

## See Also

- [[drift-protocol]] (Solana perpetual futures DEX, Entropy adopter)
- [[switchboard-vrf]] (Solana VRF oracle)
- [[orao-vrf]] (Solana multi-node VRF)

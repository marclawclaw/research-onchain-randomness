---
tags: [project, ethereum, oracle, vrf, randomness]
source: https://docs.chain.link/vrf
---

# Chainlink VRF

Chainlink VRF (Verifiable Random Function) is a provably fair, on-chain verifiable random number generator (RNG) that enables smart contracts to access random values without compromising security or usability. It is the dominant on-chain randomness solution on Ethereum and over 20 additional blockchain networks.

## Overview

For each randomness request, Chainlink VRF generates one or more random values alongside a cryptographic proof of how those values were determined. The proof is published and verified on-chain before any consuming applications can use the random output. This ensures that results cannot be tampered with or manipulated by any single entity, including oracle operators, smart contract developers, users, miners, or block builders.

Chainlink VRF seeks to address the fundamental difficulty of generating randomness on blockchains, where every node must reach the same consensus. Because random values cannot be generated natively within smart contracts, an external oracle must generate the value and return it alongside a verifiable proof.

## Architecture

### Request and Receive cycle

Chainlink VRF uses a two-phase request-and-fulfil pattern:

1. **Request phase**: A consuming contract submits a randomness request to the VRF coordinator. This is a local call that does not immediately return a result.
2. **Fulfilment phase**: Off-chain oracle nodes generate the random output and a cryptographic proof. The coordinator contract verifies the proof on-chain before releasing the random words to the consuming contract.

This two-transaction async pattern means consuming applications must handle a pending state between request and fulfilment.

### Cryptographic foundation

Chainlink VRF is based on the VRF construction described in the academic literature (Goldwasser, Micali, and Rackoff), using elliptic curve cryptography with the secp256k1 curve. Each proof consists of three components (Gamma, c, s), verified as a Schnorr-signature-like signature over the input seed and the oracle public key. The coordinator contract verifies proofs on-chain using keccak256.

### Subscription model (v2 and v2.5)

VRF v2 and v2.5 use a subscription management model. Developers create a subscription account via the Subscription Manager at vrf.chain.link and fund it with LINK tokens or native tokens. Multiple consuming contracts can be connected to a single subscription, and costs are deducted after fulfilment. This consolidates billing across many consumer contracts.

VRF v2.5 introduced the option to pay for requests in either LINK or native tokens (such as ETH), and includes an easier upgrade path to future versions.

### Direct funding model (v2.5)

Consuming contracts can alternatively pay directly for each randomness request at the time of the call, rather than through a subscription. This is more suitable for infrequent, one-off requests.

## Supported networks

Chainlink VRF is available on Ethereum mainnet and over 20 additional blockchain networks, including BNB Chain, Polygon, Avalanche, Arbitrum, Optimism, and Fantom.

## Performance characteristics

| Metric | Detail |
|---|---|
| Latency | Typically 2-3 blocks (~24-36 seconds on Ethereum mainnet) from request to fulfilment |
| Transactions per request | Two: one request transaction, one fulfilment transaction |
| Payment (Ethereum mainnet) | Approximately 0.25 LINK per request (v2.5) plus gas costs, or equivalent in native token |
| Subscription method | Subsidised via Wrapper contracts for LINK-denominated fees |

## Security model

The cryptographic proof mechanism means that the oracle operator cannot manipulate the output: only values accompanied by a valid proof are accepted by the coordinator contract. No single oracle controls the outcome because the proof must verify correctly on-chain before the random words are released.

The oracle set consists of bonded nodes that are economically incentivised to behave honestly. In the unlikely event that an adversary compromises VRF's randomness-generating secret key and obtains the ability to construct blocks on the target chain, they could strongly bias the result.

Chainlink VRF is the dominant randomness solution by adoption, securing randomness for thousands of smart contracts across Ethereum and over 20 additional blockchain networks.

> [DATA NEEDED] The original note cited "more than 2,200 unique smart contracts" but the claim cannot be verified against the cited Chainlink sources. The exact figure should be confirmed against Chainlink's official reporting or on-chain data before using in the RFP.

## Use cases

Chainlink VRF is used in applications that require unpredictable and verifiable outcomes, including:

- Blockchain gaming and NFT generation
- Random assignment of duties and resources (for example, randomly assigning judges to cases)
- Choosing a representative sample for consensus mechanisms
- Generating loot drops and unpredictable player rewards

## Limitations

- **Async two-transaction pattern**: Applications must handle a pending state between request and fulfilment, adding complexity to contract design.
- **Latency**: The 2-3 block (~24-36 second) fulfilment window makes it unsuitable for use cases requiring immediate randomness.
- **Oracle set trust**: While the cryptographic proof prevents manipulation of the output itself, consumers must still trust the bonded oracle set. A compromised secret key combined with block-building capability could bias results.
- **Cost**: Request fees (approximately 0.25 LINK per request on Ethereum mainnet v2.5, plus gas) can become significant at high request volumes.

## References

- Chainlink VRF documentation: https://docs.chain.link/vrf
- VRF v2 introduction: https://docs.chain.link/vrf/v2/introduction
- VRF v2.5 getting started: https://docs.chain.link/vrf/v2-5/getting-started
- Chainlink blog: On-Chain Verifiable Randomness: https://blog.chain.link/verifiable-random-functions-vrf-random-number-generation-rng-feature/
- VRF product page: https://chain.link/vrf

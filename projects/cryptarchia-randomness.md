---
tags: [project, logos, cryptarchia, consensus, randomness, Ouroboros]
source: https://blog.nomos.tech/nomos-cryptarchia-improving-on-ouroboros-crypsinous/
---

# Cryptarchia Randomness

## Overview

Cryptarchia is the consensus-layer randomness protocol used by the Logos blockchain (Nomos). It is a Private Proof of Stake (PPoS) protocol based on Ouroboros Crypsinous, designed specifically to provide both privacy (transaction and stake privacy) and a secure source of randomness for leader election.

The key distinguishing property of Cryptarchia over standard Ouroboros-based protocols is its use of **PVSS (Publicly Verifiable Secret Sharing)** for epoch randomness generation, rather than relying on a single leader's blockhash or a simple VRF-only approach.

## How Epoch Randomness Works

Cryptarchia divides time into **epochs**. Each epoch has three phases relevant to randomness:

1. **Commitment phase**: Each validator generates a random secret `s_i`, computes a commitment, and posts it on-chain.
2. **Reveal phase**: Validators reveal their secret shares. These are combined via PVSS to produce the epoch nonce (randomness seed) for the following epoch.
3. **Tentative nonce finalisation**: The tentative nonce at the end of the buffer period becomes the epoch nonce for `ep+1`. It is revealed at the beginning of the lottery constants finalisation period.

The epoch nonce `eta_ep` seeds the **leader election** for epoch `ep`. Each validator evaluates a **MUPRF (Multiplicative Updatable Pseudorandom Function)** seeded by their coin secret key on a string derived from `eta_ep`. The lowest VRF output wins the slot leadership. This means the randomness is used to determine who produces the next block.

## Privacy Properties

Cryptarchia's key innovation over Ouroboros Crypsinous is addressing wealth concentration. Ouroboros Crypsinous exhibits a tendency for wealth concentration toward a minority of participants because the VRF output combined with stake gives advantages to larger stakeholders. Cryptarchia mitigates this through its **stake relativisation** mechanism and PVSS-based aggregation.

Source: Nomos blog, "Wealth Concentration in PoS and Stake Relativisation" (October 2024).

## Randomness Properties

| Property | Detail |
|---|---|
| Unpredictability | The epoch nonce `eta_ep` for epoch `ep` is revealed at the start of epoch `ep`. This means the randomness for epoch `ep` is known before epoch `ep` begins (same as RANDAO). |
| Unbiasability | PVSS aggregation means no single validator can bias the output; requires a threshold of honest participants. |
| Latency | One epoch in advance (~[DATA NEEDED] seconds per epoch on Logos). |
| Leader election | Uses MUPRF evaluated on `eta_ep` to select block producers for each slot. |
| Manipulation resistance | Requires a threshold of honest validators to prevent last-revealer or equivocation attacks. |

## Comparison to Ethereum RANDAO

| Property | Ethereum RANDAO | Cryptarchia |
|---|---|---|
| Aggregation | Modular addition of validator reveals | PVSS (verifiable secret sharing) |
| Privacy | Public (BLS-signed reveals) | Private (stake and transaction privacy built in) |
| Predictability | Predictable one epoch ahead | Predictable one epoch ahead |
| Bias resistance | BLS signatures prevent equivocation | PVSS + threshold honest assumption |

## Implications for LEZ

The critical question for LEZ smart contracts is: **can the SVM execution environment access the Logos L1 randomness beacon?** If yes, LEZ contracts could use a deterministic derivation from `eta_ep` without needing an oracle. If no, LEZ contracts need an oracle solution.

This is a key architectural question that needs resolution in the LEZ design. The ideal case is direct L1 beacon access (no oracle trust assumption). The fallback is integrating an oracle (Pyth Entropy or Switchboard SRS) into the SVM environment.

**Roadmap dependency:** The LEZ roadmap includes a FURPS item (F31: Block Context) that will expose a **random oracle** to LEZ programs. See `lez_block_context.md` in the [Logos roadmap](https://roadmap.logos.co/blockchain/roadmap/lez_block_context). The checklist item "Block context exposed to programs" is still open. This random oracle (SVM `Randomness` sysvar, derived from blockhash) is the near-term mechanism for LEZ programs to access pseudorandomness. Note that this is distinct from L1 epoch nonce access: it provides SVM-level blockhash-derived randomness, not Cryptarchia PVSS consensus randomness.

**For bias-resistant L1-anchored randomness:** Once the Oracle track (AnonComms, targeting Testnet v0.2) is complete, a VRF or entropy feed could be integrated via the same oracle mechanism. Until then, an external oracle (Chainlink VRF or Pyth Entropy) is the viable path for applications requiring verifiably unbiased randomness.

## References

- Nomos blog on Cryptarchia: https://blog.nomos.tech/nomos-cryptarchia-improving-on-ouroboros-crypsinous/
- Ouroboros family: https://blog.nomos.tech/the-ouroboros-family-of-consensus-protocols-ouroboros-praos-genesis/
- Wealth concentration in Cryptarchia: https://blog.nomos.tech/wealth-concentration-in-pos-and-stake-relativisation/
- Ouroboros Crypsinous (private PoS): https://blog.nomos.tech/private-proof-of-stake-with-ouroboros-crypsinous/
- Logos genesis block spec: https://lip.logos.co/blockchain/raw/bedrock-genesis-block.html
- Logos blockchain node: https://github.com/logos-blockchain/logos-blockchain
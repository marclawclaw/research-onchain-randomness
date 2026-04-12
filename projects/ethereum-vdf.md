---
tags: [project, ethereum, vdf, consensus, randomness]
source: https://ethresear.ch/t/minimal-vdf-randomness-beacon/3566
---

# Ethereum VDF Randomness

Ethereum's original plan to augment its RANDAO randomness mechanism with Verifiable Delay Functions (VDFs) was one of the most substantive research efforts in blockchain randomness, but VDFs were ultimately not deployed in the Beacon Chain.

## The RANDAO + VDF Design

RANDAO is a simple accumulator where each block proposer contributes a BLS signature over the epoch number, and all contributions are mixed (via XOR) into a running randomness value. RANDAO is predictable: once all reveals in an epoch are known, the output is deterministic and known to all participants. This creates the "last-revealer" problem, where the final proposer in an epoch could theoretically withhold their reveal if the resulting randomness would disadvantage them.

VDFs address this by introducing a time-bound delay between the RANDAO output being formed and the randomness being used. A VDF is a function that requires a known, non-parallelisable amount of sequential computation to evaluate, but whose output is cheap to verify. The key properties (per Boneh et al., 2018) are:

- **Sequential**: even with many processors, computing f(x) requires approximately t sequential steps; no parallelism provides speedup
- **Efficiently verifiable**: given output y, any observer can verify y = f(x) in roughly log(t) time

Ethereum's minimal VDF beacon design (September 2018) split time into contiguous 8-second slots and 128-slot epochs. The VDF would be evaluated on the RANDAO output for epoch N, and the result would seed epoch N+2, introducing a two-epoch delay that prevents any single actor from influencing the randomness for the immediately following epoch.

The security argument required only one honest participant to compute and post the VDF result. Even if all VDF hardware were controlled by an attacker, they could not speed up the computation beyond the designed delay without building ASICs more than 100 times more efficient than community-provided hardware.

## Why VDFs Were Not Deployed

Ethereum ultimately shipped the Beacon Chain (Phase 0, December 2020) without VDFs. The decision was driven by several compounding factors:

1. **ASIC development burden**: The Ethereum Foundation funded VDF hardware research through the VDF Alliance (vdfresearch.org), but producing application-specific integrated circuits that could perform the required repeated modular squaring in hidden-order groups proved slow and costly. The team estimated that trustworthy, open-source VDF hardware would require significant investment with no direct economic incentive for operators to run it.

2. **Trustless setup complexity**: Secure VDFs require a group of unknown order (typically RSA-style or class groups of imaginary quadratic fields). Generating such a group without a trusted party who knows the order was a active research problem. The Diogenes protocol (2020) made progress on multi-party computation for RSA modulus generation with a dishonest majority, but the engineering complexity remained high.

3. **BLS signatures were sufficient**: Switching from hash onions to BLS signatures for RANDAO contributions (EIP-2333 / BLS12-381) delivered most of the desired properties without any additional hardware. The BLS-based RANDAO is simple, verifiable, and benefits from aggregation properties that enable distributed validator technology.

4. **Staggered deployment was complex**: A fully functional VDF beacon required careful timing across all validators. Any clock skew could cause the VDF output to arrive too late for the target epoch, breaking the protocol.

The conclusion from the Ethereum research community was that the added security benefit of VDFs did not justify the deployment complexity at that stage of the protocol's evolution. The RANDAO + BLS approach was considered adequate for the threat model at launch.

## Polkadot's VDF Approach

Polkadot also recognised the value of VDFs for randomness. According to the Polkadot developer documentation, their randomness combines the VRF output from block production (BABE) with a VDF computed over the VRF results from two epochs prior. This ensures that even if an attacker could predict VRF outputs, they cannot determine the final randomness until the VDF computation completes. Polkadot notes that VDF hardware (specialised ASICs) would be required for this, and while only one honest device is needed to secure the system, the cost and complexity of running VDF devices without direct economic incentives is a significant friction point.

## Solana's Hash Chain (Not a True VDF)

Solana uses a SHA-256 hash chain as part of its Proof of History mechanism, which the Solana documentation previously described as a "verifiable delay function." However, this is not a true VDF in the cryptographic sense. As noted in Solana GitHub Issue #388, a SHA-256 hash chain takes the same amount of time to verify as it does to compute. A true VDF requires verification to be exponentially faster than computation (roughly log(t) versus t steps). Solana's hash chain is better described as a proof of sequential work rather than a VDF. The Solana team acknowledged this distinction in their documentation, noting that the authors of the original VDF paper would object to the term being applied to their approach.

## VDF Research Landscape

The VDF Alliance (vdfresearch.org) catalogued extensive research from 2018-2022. Key constructions include:

- **Pietrzak (2018)**: Repeated squaring in groups of unknown order, based on the low-order assumption
- **Wesolowski (2018)**: Independent construction using similar repeated-squaring techniques with a different verification scheme
- **Boneh, Bünz, Fisch (2018)**: The original VDF paper, using injective rational maps (termed a "weak VDF" due to some parallelisability in evaluation)

Security of Pietrzak/Wesolowski schemes relies on the low-order assumption in the chosen group. RSA groups require careful construction to eliminate low-order elements (using strong primes and quotienting by {1, -1}). Trustless setup of such groups remained an active research problem, with contributions from Diogenes (2020) and others working toward multi-party computation approaches that avoid any single party knowing the group order.

## Sources

- https://ethresear.ch/t/minimal-vdf-randomness-beacon/3566
- https://ethresear.ch/t/verifiable-delay-functions-and-attacks/2365
- https://blog.trailofbits.com/2018/10/12/introduction-to-verifiable-delay-functions-vdfs/
- https://vdfresearch.org/
- https://docs.polkadot.com/reference/parachains/randomness/
- https://github.com/solana-labs/solana/issues/388
- https://eth2book.info/latest/part2/building_blocks/randomness/
- https://www.adevarlabs.com/blog/on-chain-randomness-on-solana-predictability-manipulation-safer-alternatives-part-1
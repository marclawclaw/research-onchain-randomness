---
tags: [project, ethereum, consensus, randao, randomness]
source: https://eth2book.info/latest/part2/building_blocks/randomness/
---

# Ethereum RANDAO

RANDAO is Ethereum's in-protocol randomness mechanism, operating at the consensus layer of the Beacon Chain. It accumulates entropy from block proposers to assign critical duties: block proposal, committee membership, and sync committee participation. The mechanism exists because predictability provides attackers with exploitable opportunities, such as targeted denial-of-service attacks against future proposers, committee bribery, or transaction censorship (eth2book, 2025).

## Mechanism

RANDAO is a commit-reveal scheme based on CRS (Commit-Reveal Scheme). In CRS, each member commits a value to a group and the final number is a combination of those values (Tran & Quang, 2024). In Ethereum 2.0, the mechanism works as follows.

Each block contains a `randao_reveal` field. The proposer signs the current epoch number using its BLS private key, and this signature becomes the reveal. The signature is verified against the proposer's public key before being mixed in, which means the proposer has almost no choice about its contribution: it either contributes the correct signature or withholds its block (eth2book, 2025).

The mix operation is defined as:

```
mix = xor(get_randao_mix(state, epoch), hash(body.randao_reveal))
```

The hash of the signature is used to reduce the 762-bit compressed BLS signature to 256 bits. The XOR operation provides useful cryptographic properties and, critically, is commutative, which slightly reduces the attacker's ability to grind randomness across epochs compared to a non-commutative hash operation (eth2book, 2025). The Beacon Chain maintains the current RANDAO value in the `randao_mixes` field of the beacon state, and past values at epoch boundaries are also stored to allow historical committee assignments to be recalculated for slashing purposes.

### Hash onion (deprecated)

The original RANDAO design used a hash onion. Before joining the Beacon Chain, a validator would generate a random number and submit the result of repeatedly hashing it thousands of times as a commitment. When proposing a block, the reveal would be the pre-image of that commitment, peeling off one layer. This was viable but clunky in practice, particularly because orphaned blocks would expose a proposer's reveal (eth2book, 2025).

### BLS signatures (current)

The current design replaced hash onions with BLS signatures. Every validator already has a secret key used for signing blocks and attestations; the signatures are uniformly random and satisfy both unpredictability (unknown to other validators) and verifiability (easily checked against the public key). The aggregation property of BLS signatures also enables Distributed Validator Technology, which would have been difficult with the hash onion approach (eth2book, 2025).

## Vulnerabilities

### Last Revealer Attack (LRA)

In a naive CRS-based RANDAO using XOR aggregation, the last participant to reveal can gain control over the final output. If the attacker is the proposer in the final slot, they know the accumulated value from all previous slots and can choose whether to reveal or withhold their signature. By selectively revealing or withholding, the attacker can bias the output toward a value that favours them (Tran & Quang, 2024). In the original Ethereum 1.0 RANDAO contract, Buterin (2018, cited in Tran & Quang, 2024) showed that if an attacker controlled 36% of total staked ETH, they could potentially manipulate proposer selection.

Critically, the XOR aggregation means that `N xor N = 0` and `0 xor N = N`. If colluding validators duplicate each other's inputs, the honest contributions can be cancelled out. With N/2 colluding validators, the output can be coerced to zero; with N/2 + 1, it can be coerced to an arbitrary value (Revelry, 2018). Ethereum 2.0 mitigates this by requiring each validator to contribute only the correct BLS signature over the epoch number, which they cannot choose or duplicate.

### RANDAOtage

RANDAOtage is an attack vector combining RANDAO bias with validator outages. An attacker who can take down a sufficient number of validators (through coordinated censorship or infrastructure attacks) can monopolise beacon block production and bias the RANDAO output. This is a committee-corruption attack that exploits the interplay between network reliability and randomness quality (notes.ethereum.org).

### Predictability

The RANDAO output for a given epoch is known before the epoch begins, once the reveal phase for the previous epoch is complete. It is predictable but not manipulable by individual actors due to the economic and cryptographic constraints described above. The lookahead is limited to two epochs, which constrains but does not eliminate the window for attack (eth2book, 2025).

## Proposed fixes

### BLS integration (implemented)

Ethereum 2.0 addresses the LRA by integrating BLS signatures into the protocol. Because each proposer's contribution is a deterministic cryptographic signature over the epoch number, validators cannot choose alternative values and cannot collude to cancel each other out in the same way as with a naive XOR-based CRS. Vitalik Buterin confirmed that BLS integration negates the LRA (Revelry, 2018).

### Verifiable Delay Functions (VDF)

Researchers proposed VDF (Verifiable Delay Function) as a solution to prevent any validator from computing the final random number before the reveal phase is complete. The Ethereum Foundation confirmed a minimal VDF (mVDF) version for use after Phase 2, though it requires specialised hardware (Tran & Quang, 2024).

### Shamir's Secret Sharing (SSS)

An alternative academic proposal uses Shamir's Secret Sharing to construct a RANDAO scheme that prevents LRA under favourable network conditions. SSS-based RANDAO provides a controllable security level for random proposer selection (Tran & Quang, 2024). This approach has not been implemented in the Ethereum protocol.

## Sources

- eth2book.info. (2025). *Randomness*. https://eth2book.info/latest/part2/building_blocks/randomness/
- Tran, T.T.T. & Quang, L.M. (2024). *RANDAO-based RNG: Last Revealer Attacks in Ethereum 2.0*. arXiv:2403.09541. https://arxiv.org/html/2403.09541v1/
- Revelry. (2018). *Defeating Crypto's Favourite Random Number Generator*. https://revelry.co/insights/blockchain/critical-randao-vulnerability/
- Ethereum Research. (n.d.). *RANDAOtage: committee corruption from RANDAO bias and validator outages*. https://notes.ethereum.org/iMxxlEkuQMiPkEL1S6SfbQ

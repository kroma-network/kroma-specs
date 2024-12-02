# Differences from Optimism

<!-- All glossary references in this file. -->

[g-l2-output-root]: ../glossary.md#l2-output-root
[g-zk-fault-proof]: ../glossary.md#zk-fault-proof

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Nodes](#nodes)
  - [Verifier -> Validator](#verifier---validator)
  - [Compositions](#compositions)
- [Validator](#validator)
  - [ZK fault proof](#zk-fault-proof)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Nodes

There are two types of network participants in the OP Stack:

- [Sequencers](https://specs.optimism.io/background.html#sequencers) consolidate
 users' on/off chain transactions into blocks. They submit checkpoint outputs as well as batch transactions.
- [Verifiers](https://specs.optimism.io/background.html#verifiers) verify rollup
   integrity and dispute invalid assertions.

It is crucial to have at least one honest verifier who can verify the integrity of the rollup chain to ensure the
ongoing security of the network. However, there exists a well-known obstacle known as the 'Verifier's Dilemma' that
poses a threat to the security of optimistic rollups by introducing disincentives in such scenarios.

To resolve the 'Verifier's Dilemma', we have devised an incentive mechanism that motivates node operators to actively
participate in the Kroma network. As part of this redesign, we have separated the responsibility of submitting
checkpoint outputs from `sequencers` and assigned it to `verifiers`. As a result, these participants have been renamed
to reflect these role changes and the future direction of Kroma's decentralization. For more detailed information about
our decentralization scheme, please refer to
[this article](https://medium.com/@kroma-network/the-road-to-kromas-decentralization-38f8e46df442)
on the Kroma blog.

### Verifier -> Validator

We utilize the term `validator` to denote a participant who is responsible for submitting the
[L2 output root][g-l2-output-root] and validating its accuracy by either submitting dispute challenges (during the
optimistic rollup phase) or providing ZK validity proofs (during the ZK rollup phase). This concept bears a resemblance
to how L1 validators cast FFG votes at each epoch.

### Compositions

Kroma maintains the modular architecture of the OP Stack, with various components communicating through Json RPC calls.
As part of the transition from `verifier` to `validator`, we have renamed the `proposer` (op-proposer) component of the
OP Stack to `validator` (kroma-validator) and made necessary modifications to the code to handle the dispute challenge
processes.

The followings are components that are used to run different types of nodes:

| Node        | Components                                                           |
|-------------|----------------------------------------------------------------------|
| `Sequencer` | `L2 EL client` + `L2 CL client` + `kroma-batcher`                    |
| `Validator` | `L2 EL client` + `L2 CL client` + `kroma-validator` + `kroma-prover` |
| `Full node` | `L2 EL client` + `L2 CL client`                                      |

**NOTE:** Here `L2 EL client` means `kroma-geth` and `L2 CL client` means `kroma-node`. `L2 EL client` can
be expanded to other clients for pragmatic decentralization.

## Validator

### ZK fault proof

Instead of [cannon], Kroma uses zkVM for [ZK fault proof][g-zk-fault-proof].

[cannon]: https://github.com/ethereum-optimism/cannon

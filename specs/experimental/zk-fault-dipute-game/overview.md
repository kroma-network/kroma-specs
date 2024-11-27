# ZK Fault Dispute Game
<!-- All glossary references in this file. -->

[g-zk-fault-proof]: ../../glossary.md#zk-fault-proof
[g-withdrawals]: ../../glossary.md#withdrawals

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [ZK Fault Dispute Game Creation](#zk-fault-dispute-game-creation)
- [Challenge Creation](#challenge-creation)
- [Dissection](#dissection)
- [Proving Fault using zkVM](#proving-fault-using-zkvm)
  - [ZK Fault Proof](#zk-fault-proof)
  - [zkVM Proving System](#zkvm-proving-system)
- [Resolution of ZK Fault Dispute Game](#resolution-of-zk-fault-dispute-game)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The ZK Fault Dispute Game (ZKFDG) is a new type of dispute game compatible with the OP Stack. In ZKFDG, the first
disagreeing output root between defender and challenger is found by the process called "dissection". Then, both the
derivation of L2 block corresponding to that output root and the execution of the block's transactions are carried out
in zkVM. The validity of the execution is guaranteed by the ZK proof, which can be verified on-chain.

The main difference from [Kroma's previous ZK Fault Proof System](../../zk-fault-proof/challenge.md) is that ZKFDG uses
a "zkVM" instead of a "zkEVM" to handle the proving fault process. By using zkVM for fault proof, it is possible to
verify the entire processes from derivation to execution without any additional developments of ZK circuit.

ZKFDG implements the OP Stack's `IDisputeGame`, implying that it is fully compatible as one of the OP Stack's dispute
game type. This also implies that it can be applied to the OP Stack's multi-proof system in the future.

## ZK Fault Dispute Game Creation

The ZK Fault Dispute Game is created by a validator selected by the
[`ValidatorManager`](../../protocol/validator-v2/validator-manager.md) at intervals defined as the
`SUBMISSION_INTERVAL` configured in the [`L2OutputOracle`](../../protocol/validation.md#l2-output-oracle-smart-contract)
contract. The validator can create a ZKFDG-type dispute game through the `DisputeGameFactory` with an extra data. The
extra data is composed as follows:

```text
extra_data = current_l2_block_num ++ prev_game_proxy

current_l2_block_num = uint256
prev_game_proxy      = bytes32
```

When the ZKFDG is created by the validator, the followings are verified:

- the validator should be eligible to submit a claim
- The `current_l2_block_num` should be the next block number for submission of root claim (output root)
- The `prev_game_proxy` should be the right previous ZKFDG contract

If all validations are passed, a new ZKFDG contract is created. The address of this ZKFDG contract is stored with
current L2 block number on the L2OutputOracle contract, which can be used for the creation of the next ZKFDG. The root
claim stored in `prev_game_proxy` is used as the starting output root for the dissection process to find the first
disagreeing block.

## Challenge Creation

As with Kroma's [previous Fault Proof System][prev-challenge-creation], a challenge can be initiated by a validator who
disagrees the submitted claim. The challenge process begins by submitting the intermediate segments between starting
output root and disputed output root by challenger.

[prev-challenge-creation]: ../../zk-fault-proof/challenge.md#challenge-creation

```solidity
    /**
     * @notice Creates a challenge against an invalid claim.
     *
     * @param _l1BlockHash   The block hash of L1 at the time the output L2 block was created.
     * @param _l1BlockNumber The block number of L1 with the specified L1 block hash.
     * @param _segments      Array of the segments. A segment is the first output root of a specific range.
     */
    function createChallenge(
        bytes32 _l1BlockHash,
        uint256 _l1BlockNumber,
        bytes32[] calldata _segments
    ) external;
```

## Dissection

The dissection process is carried out in the same manner as in Kroma's [previous Fault Proof System][prev-bisection].
Through interactions between the challenger and the defender (the game creator), the first disagreeing output root can
be specified.

[prev-bisection]: ../../zk-fault-proof/challenge.md#bisection

```solidity
    /**
     * @notice Selects an invalid section and submits intermediate segments of that section.
     *
     * @param _challenger  Address of the challenger.
     * @param _pos         Position of the last valid segment.
     * @param _segments    Array of the segments. A segment is the first output root of a specific range.
     */
    function dissect(
        address _challenger,
        uint256 _pos,
        bytes32[] calldata _segments
    ) external;
```

## Proving Fault using zkVM

Once the first disagreeing output root is specified, the challenger can prove that the disputed claim is incorrect using
a ZK fault proof.

### ZK Fault Proof

A [ZK fault proof][g-zk-fault-proof] demonstrates that a state transition from S to S’ is valid based on the
transactions within the block. While this may seem similar to a validity proof, the key difference lies in its
purpose. The ZK fault proof is used to demonstrate that a state transition from S to S’’ is incorrect by
providing evidence of a valid state transition from S to S’.

### zkVM Proving System

See [zkVM Prover](../../zk-fault-proof/zkvm-prover.md#zkvm-proving-system) for details.

## Resolution of ZK Fault Dispute Game

For the ZKFDG to be resolved, the `FINALIZATION_PERIOD` must pass after it is created. After the `FINALIZATION_PERIOD`
has passed, the status of dispute game can be determined based on the presence of the root claim in the dispute game. If
a non-zero value is stored for the root claim, the dispute game's status will be resolved as `DEFENDER_WINS`. If the
root claim value is stored with zero, it indicates that the challenger succeeded in zk proving for fault, thus the
dispute game will be resolved as `CHALLENGER_WINS`. Only ZKFDG resolved as `DEFENDER_WINS` can be used for proving and
finalizing [withdrawals][g-withdrawals] to L1.

# ZK Fault Dispute Game

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [ZK Fault Dispute Game Creation](#zk-fault-dispute-game-creation)
- [Challenge Creation](#challenge-creation)
- [Dissection](#dissection)
- [Proving Fault using ZKVM](#proving-fault-using-zkvm)
- [Resolution of ZK Fault Dispute Game](#resolution-of-zk-fault-dispute-game)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

[g-withdrawal]: ../../glossary.md#withdrawal

## Overview

The ZK Fault Dispute Game (ZKFDG) is a new type of dispute game compatible with the OP Stack. In ZKFDG, the first
disagreeing output root is found by the process called "dissection". Then, both derivation of L2 block from L1
corresponding to that output root and executing the block's transactions is executed in zkVM. The validity of the
execution is guaranteed by the ZK proof, which can be verified on-chain.

The main difference from [Kroma's previous ZK Fault Proof System](../../fault-proof/challenge.md) is that ZKFDG uses a
"zkVM" instead of a "zkEVM" to handle all processes. By using zkVM for fault proof, it is enabled to verify all
processes from derivation to execution without any additional developments of ZK circuit.

ZKFDG implements the OP Stack's `IDisputeGame`, implying that it is fully compatible with the OP Stack's dispute game.
This also implies that it can be applied to the OP Stack's multi-proof system in the future.

## ZK Fault Dispute Game Creation

The ZK Fault Dispute Game is created by a validator selected by the
[`ValidatorManager`](../../protocol/validator-v2/validator-manager.md) at intervals defined by the
`SUBMISSION_INTERVAL` configured in the [`L2OutputOracle`](../../protocol/validation.md#l2-output-oracle-smart-contract)
contract. The validator can create a ZKFDG-type dispute game through the `DisputeGameFactory` with an extra data. The
extra data is composed as follows:

```text
extra_data = current_l2_block_num ++ prev_game_proxy

current_l2_block_num = uint256
prev_game_proxy      = bytes32
```

When the ZKFDG is created by the validator, it is verified that the validator is eligible to submit a claim and the
`prev_game_proxy` is valid ZKFDG contract. The `prev_game_proxy` is used as the starting output root for the dissection
process to find the first disagreeing block. If all validations pass, the address of this ZKFDG contract is stored with
current L2 block number on the L2OutputOracle contract. This will be used for the creation of the next ZKFDG.

## Challenge Creation

As with Kroma's previous Fault Proof System, a challenge can be initiated by a validator who disagrees the submitted
claim. The challenge process begins by submitting the initial segments by challenger.

```solidity
/**
     * @notice Creates a challenge against an invalid claim.
     *
     * @param _l1BlockHash   The block hash of L1 at the time the output L2 block was created.
     * @param _l1BlockNumber The block number of L1 with the specified L1 block hash.
     * @param _segments      Array of the segment. A segment is the first output root of a specific range.
     */
    function createChallenge(
        bytes32 _l1BlockHash,
        uint256 _l1BlockNumber,
        bytes32[] calldata _segments
    ) external;
```

## Dissection

The dissection process is carried out in the same manner as in Kroma's previous Fault Proof System. Through interactions
between the challenger and the defender (the game creator), the first disagreeing output root can be specified.

```solidity
    /**
     * @notice Selects an invalid section and submit segments of that section.
     *
     * @param _challenger  Address of the challenger.
     * @param _pos         Position of the last valid segment.
     * @param _segments    Array of the segment. A segment is the first output root of a specific range.
     */
    function dissect(
        address _challenger,
        uint256 _pos,
        bytes32[] calldata _segments
    ) external;
```

## Proving Fault using ZKVM

## Resolution of ZK Fault Dispute Game

For the ZKFDG to be resolved, the `FINALIZATION_PERIOD` must pass after it is created. After the `FINALIZATION_PERIOD`
has passed, the status of dispute game can be determined based on the presence of the claim root in the dispute game. If
a non-zero value is stored for the claim root, the dispute game's status will be resolved as `DEFENDER_WINS`. If the
claim root value is stored with zero, it indicates that the challenger succeeded in zk proving for fault, and the
dispute game will be resolved as `CHALLENGER_WINS`. The resolved ZKFDG is used for [withdrawal][g-withdrawal] to L1.

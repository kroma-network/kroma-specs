# ZK Fault Dispute Game
<!-- All glossary references in this file. -->

[g-zk-fault-proof]: ../../glossary.md#zk-fault-proof
[g-l2-chain-derivation]: ../../glossary.md#l2-chain-derivation
[g-sequencer-batch]: ../../glossary.md#sequencer-batch
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
    - [Guest Program](#guest-program)
    - [Host Program](#host-program)
    - [Public Values of Proof](#public-values-of-proof)
    - [`ZkVerifier` interface](#zkverifier-interface)
- [Resolution of ZK Fault Dispute Game](#resolution-of-zk-fault-dispute-game)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The ZK Fault Dispute Game (ZKFDG) is a new type of dispute game compatible with the OP Stack. In ZKFDG, the first
disagreeing output root between defender and challenger is found by the process called "dissection". Then, both the
derivation of L2 block corresponding to that output root and the execution of the block's transactions are carried out
in zkVM. The validity of the execution is guaranteed by the ZK proof, which can be verified on-chain.

The main difference from [Kroma's previous ZK Fault Proof System](../../fault-proof/challenge.md) is that ZKFDG uses a
"zkVM" instead of a "zkEVM" to handle the proving fault process. By using zkVM for fault proof, it is possible to verify
the entire processes from derivation to execution without any additional developments of ZK circuit.

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

[prev-challenge-creation]: ../../../specs/fault-proof/challenge.md#challenge-creation

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

[prev-bisection]: ../../../specs/fault-proof/challenge.md#bisection

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

### ZK Fault Proof

A [ZK fault proof][g-zk-fault-proof] demonstrates that a state transition from S to S’ is valid based on the
transactions within the block. While this may seem similar to a validity proof, the key difference lies in its
purpose. The ZK fault proof is used to demonstrate that a state transition from S to S’’ is incorrect by
providing evidence of a valid state transition from S to S’.

### zkVM Proving System

The zkVM (Zero-Knowledge Virtual Machine) is a virtual machine that executes guest program compiled with a specified
compiler generating zero-knowledge proofs for their execution. The guest program can be written in standard programming
languages, such as Rust and Go.

#### Guest Program

For ZK Fault Dispute Game, the guest program is an extension of [L2 Chain Derivation][g-l2-chain-derivation] that
includes a connectivity check among the blocks from L1 origin block to the specified block `C`. The hash of Block
`C` is determined as the parent hash stored when the ZK Fault Dispute Game is created by calling `create()` of
the [Dispute Game Factory].

If the attacker manipulates any data within the extension of L2 Chain Derivation, it will affect the block hash `C`.
This is because all data is ultimately linked to the block hash `C` through the hash chain. Therefore if any data
manipulating attack can be thwarted by checking the block hash `C` value. For example, if an attacker creates a proof
using a transaction that is not included in the [sequencer batch][g-sequencer-batch], the block `C` value will change,
preventing them from winning the challenge.

[Dispute Game Factory]: https://github.com/ethereum-optimism/specs/blob/46d411bfea922c520a1d43329dbf78a2f6966ae0/specs/fault-proof/stage-one/dispute-game-interface.md#disputegamefactory-interface

#### Host Program

The host program is a main part of the prover, responsible for delivering the guest program and its input to the zkVM.
The host program first executes the guest program to gather the necessary data, which is same as the data stored in
[PreImageOracle] during [Optimism's Fault Dispute Game]. For ZKFDG, however, additional data is required, which is the
L1 block data from the origin of target L2 block to block `C`. Finally, the host program delivers the guest program
and the preimages to the zkVM to obtain the zkVM proof.

[PreImageOracle]: https://github.com/ethereum-optimism/specs/blob/46d411bfea922c520a1d43329dbf78a2f6966ae0/specs/fault-proof/stage-one/fault-dispute-game.md#preimageoracle
[Optimism's Fault Dispute Game]: https://github.com/ethereum-optimism/specs/blob/46d411bfea922c520a1d43329dbf78a2f6966ae0/specs/fault-proof/stage-one/fault-dispute-game.md#fault-dispute-game

#### Public Values of Proof

To mark which blocks have been executed, the proof publicly reveals the following data.

``` plain
1. Output root at the parent block of the target
2. Output root at the target block
3. Hash of the block C
```

#### `ZkVerifier` interface

The proof generated by a prover can be verified on a verifier contract that includes the following interface.
The verification in a challenge process will be implemented using the verify function provided by the verifier contract.

``` solidity
interface ZKVerifier {
    /// @notice The entrypoint for verifying the proof of zk fault proof.
    /// @param _publicValues The encoded public values.
    /// @param _proofBytes The encoded proof.
    /// @return parentOutputRoot The output root at the parent block of the target.
    /// @return outputRoot The output root at the target block.
    /// @return l1Head The block hash of the l1 block stored in ZK fault dispute game contract.
    function verify(
      bytes calldata _publicValues, 
      bytes calldata _proofBytes
    ) public view returns (
      bytes32 parentOutputRoot, 
      bytes32 outputRoot, 
      bytes32 l1Head
    );
}
```

## Resolution of ZK Fault Dispute Game

For the ZKFDG to be resolved, the `FINALIZATION_PERIOD` must pass after it is created. After the `FINALIZATION_PERIOD`
has passed, the status of dispute game can be determined based on the presence of the root claim in the dispute game. If
a non-zero value is stored for the root claim, the dispute game's status will be resolved as `DEFENDER_WINS`. If the
root claim value is stored with zero, it indicates that the challenger succeeded in zk proving for fault, thus the
dispute game will be resolved as `CHALLENGER_WINS`. Only ZKFDG resolved as `DEFENDER_WINS` can be used for proving and
finalizing [withdrawals][g-withdrawals] to L1.

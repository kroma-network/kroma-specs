# Validations

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Submitting L2 Output Commitments](#submitting-l2-output-commitments)
- [L2 Output Commitment Construction](#l2-output-commitment-construction)
  - [Output Payload(Version 0)](#output-payloadversion-0)
- [L2 Output Oracle Smart Contract](#l2-output-oracle-smart-contract)
  - [Configuration of L2OutputOracle](#configuration-of-l2outputoracle)
- [Security Considerations](#security-considerations)
  - [L1 Reorgs](#l1-reorgs)
- [Summary of Definitions](#summary-of-definitions)
  - [Constants](#constants)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->

[g-l1]: ../glossary.md#layer-1-l1
[g-l2]: ../glossary.md#layer-2-l2
[g-zk-fault-proof]: ../glossary.md#zk-fault-proof
[g-mpt]: ../glossary.md#merkle-patricia-trie
[g-priority-round]: ../glossary.md#priority-round
[g-public-round]: ../glossary.md#public-round

## Overview

![Validation Overview](../static/assets/verifier-proving-fault-proof.svg)

After processing one or more blocks, the outputs will need to be synchronized with [L1][g-l1] for trustless execution of
L2-to-L1 messaging, such as withdrawals.
These output proposals act as the bridge's view into the L2 state.
Actors called "Validators" submit the output roots to L1 and can be contested with a [ZK fault proof][g-zk-fault-proof],
with a bond at stake if the proof is wrong.

## Submitting L2 Output Commitments

> **_NOTE:_**
> Before the introduction of the KRO governance token, the
> [validator system was ETH-based](../deprecated/validator-v1/validator-pool.md). With introduction of KRO token, the
> system had been transited to [KRO-based validator system](./validator-v2/overview.md), mitigating several limitations
> of the ETH-based system.

The validator's role is to construct and submit output roots, which are commitments to the L2's state, to the
`L2OutputOracle` contract on L1. To do this, the validator periodically queries the [rollup node](rollup-node.md) for
the latest output root derived from the latest [finalized][finality] L1 block. It then takes the output root and submits
it to the `L2OutputOracle` contract on L1.

[finality]: https://hackmd.io/@prysmaticlabs/finality

The validator that submits the output root is determined by the validator system.
The output submission rounds are divided into [Priority Round][g-priority-round] and [Public Round][g-public-round],
and the time limit of each round is configured as `ROUND_DURATION` in the validator contract.
A prioritized validator is selected randomly by validator system, and the prioritized validator must submit output
within the `Priority Round` time. If the prioritized validator fails to submit within the `Priority Round`, the round
moves to the `Public Round`, where all validators can submit output regardless of priority.

The validator is expected to submit output roots on a deterministic
interval based on the configured `SUBMISSION_INTERVAL` in the `L2OutputOracle`. The larger
the `SUBMISSION_INTERVAL`, the less often L1 transactions need to be sent to the `L2OutputOracle`
contract, but L2 users will need to wait a bit longer for an output root to be included in L1
that includes their intention to withdrawal from the system.

The honest `kroma-validator` algorithm assumes a connection to the `L2OutputOracle` contract to know
the L2 block number that corresponds to the next output root that must be submitted. It also
assumes a connection to an `op-node` to be able to query the `kroma_syncStatus` RPC endpoint.

Once your submitted output is [finalized][finality], the submitter becomes eligible for a reward.
For more information on this, see [Validation Rewards](./validator-v2/validator-manager.md#reward-distribution).

```python
import time

while True:
    next_checkpoint_block = L2OutputOracle.nextBlockNumber()
    rollup_status = kroma_node_client.sync_status()
    if rollup_status.finalized_l2.number >= next_checkpoint_block:
        output = kroma_node_client.output_at_block(next_checkpoint_block)
        tx = send_transaction(output)
    time.sleep(poll_interval)
```

## L2 Output Commitment Construction

The `output_root` is a 32bytes string, which is derived based on a versioned scheme:

```pseudocode
output_root = keccak256(version_byte || payload)
```

where:

1. `version_byte` (`bytes32`) a simple version string which increments anytime the construction of the output root
   is changed.

2. `payload` (`bytes`) is a byte string of arbitrary length.

### Output Payload(Version 0)

The version 0 payload is defined as:

```pseudocode
payload = state_root || withdrawal_storage_root || block_hash
```

where:

1. The `block_hash` (`bytes32`) is the block hash for the [L2][g-l2] block that the output is generated from.

2. The `state_root` (`bytes32`) is the [Merkle Patricia Trie][g-mpt] root of all execution-layer accounts.
   This value is frequently used and thus elevated closer to the L2 output root, which removes the need to prove its
   inclusion in the pre-image of the `block_hash`. This reduces the merkle proof depth and cost of accessing the
   L2 state root on L1.

3. The `withdrawal_storage_root` (`bytes32`) elevates the Merkle Trie root of the
   [L2ToL1MessagePasser contract](withdrawals.md#the-l2tol1messagepasser-contract) storage. Instead of making a
   [MPT][g-mpt] proof for a withdrawal against the state root (proving first the storage root of the
   L2toL1MessagePasser against the state root, then the withdrawal against that storage root), we can prove against the
   L2toL1MessagePasser's storage root directly, thus reducing the verification cost of withdrawals on L1.

The height of the block where the output is submitted has been delayed by one. Note that `next_block_hash` is deprecated
and not used in the construction of the output root after [Kroma MPT migration fork](./mpt-migration/overview.md).

## L2 Output Oracle Smart Contract

L2 blocks are produced at a constant rate of `L2_BLOCK_TIME`.
A new L2 output MUST be appended to the chain once per `SUBMISSION_INTERVAL` which is based on the number of L2 blocks.

L2 Output Oracle Smart Contract implements the following interface:

```solidity
interface L2OutputOracle {
    event OutputSubmitted(
        bytes32 indexed outputRoot,
        uint256 indexed l2OutputIndex,
        uint256 indexed l2BlockNumber,
        uint256 l1Timestamp
    );

    event OutputReplaced(
        uint256 indexed outputIndex,
        address indexed newSubmitter,
        bytes32 newOutputRoot
    );

    function replaceL2Output(
        uint256 _l2OutputIndex,
        bytes32 _newOutputRoot,
        address _submitter
    ) external;

    function submitL2Output(
        bytes32 _outputRoot,
        uint256 _l2BlockNumber,
        bytes32 _l1BlockHash,
        uint256 _l1BlockNumber
    ) external payable;
}
```

### Configuration of L2OutputOracle

The `startingBlockNumber` must be at least the number of the first recorded L2 block.
The `startingTimestamp` MUST be the same as the timestamp of the first recorded L2 block.

Thus, the first `outputRoot` submitted will be at height `startingBlockNumber`, and each subsequent one will be at
height incremented by `SUBMISSION_INTERVAL`.

## Security Considerations

### L1 Reorgs

If the L1 has a reorg after an output has been generated and submitted, the L2 state and correct output may change
leading to a misbehavior. This is mitigated against by allowing the validator to submit an
L1 block number and hash to the [L2 Output Oracle](#l2-output-oracle-smart-contract) when appending a new output;
in the event of a reorg, the block hash will not match that of the block with that number and the call will revert.

## Summary of Definitions

### Constants

| Name                   | Value  | Unit           |
|------------------------|--------|----------------|
| `SUBMISSION_INTERVAL`  | `1800` | blocks         |
| `L2_BLOCK_TIME`        | `2`    | seconds        |
| `ROUND_DURATION`       | `30`   | minutes        |

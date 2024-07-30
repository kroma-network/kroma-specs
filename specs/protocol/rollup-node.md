# Rollup Node Specification

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Driver](#driver)
  - [Derivation](#derivation)
- [L2 Output RPC method](#l2-output-rpc-method)
  - [Structures](#structures)
    - [BlockID](#blockid)
    - [L1BlockRef](#l1blockref)
    - [L2BlockRef](#l2blockref)
    - [SyncStatus](#syncstatus)
  - [Output Method API](#output-method-api)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->

[g-block]: ../glossary.md#block
[g-derivation]: ../glossary.md#L2-chain-derivation
[g-exec-engine]: ../glossary.md#execution-engine
[g-l1]: ../glossary.md#layer-1-l1
[g-l2]: ../glossary.md#layer-2-l2
[g-payload-attr]: ../glossary.md#payload-attributes
[g-sequencer-batch]: ../glossary.md#sequencer-batch
[g-receipts]: ../glossary.md#receipt
[g-reorg]: ../glossary.md#re-organization
[g-rollup-driver]: ../glossary.md#rollup-driver
[g-rollup-node]: ../glossary.md#rollup-node

## Overview

The [rollup node][g-rollup-node] is the component responsible for [deriving the L2 chain][g-derivation] from [L1][g-l1]
blocks (and their associated [receipts][g-receipts]).

The part of the rollup node that derives the [L2][g-l2] chain is called the [rollup driver][g-rollup-driver]. This
document is currently only concerned with the specification of the rollup driver.

## Driver

The task of the [driver][g-rollup-driver] in the [rollup node][g-rollup-node]
is to manage the [derivation][g-derivation] process:

- Keep track of L1 head block
- Keep track of the L2 chain sync progress
- Iterate over the derivation steps as new inputs become available

### Derivation

This process happens in three steps:

1. Select inputs from the L1 chain, on top of the last L2 block:
   a list of blocks, with transactions and associated data and receipts.
2. Read L1 information, deposits, and [sequencing batches][g-sequencer-batch] in order to generate
   [payload attributes][g-payload-attr] (essentially [a block without output properties][g-block]).
3. Pass the payload attributes to the [execution engine][g-exec-engine], so that the L2 block (including [output block
   properties][g-block]) may be computed.

While this process is conceptually a pure function from the L1 chain to the L2 chain, it is in practice incremental. The
L2 chain is extended whenever new L1 blocks are added to the L1 chain. Similarly, the L2 chain re-organizes whenever the
L1 chain [re-organizes][g-reorg].

For a complete specification of the L2 block derivation, refer to the [L2 block derivation document](./derivation.md).

## L2 Output RPC method

The Rollup node has its own RPC method, `optimism_outputAtBlock` which returns a 32
byte hash corresponding to the [L2 output root](validation#l2-output-commitment-construction).

### Structures

These define the types used by rollup node API methods.
The types defined here are extended from the [engine API specs][engine-structures].

#### BlockID

- `hash`: `DATA`, 32 Bytes
- `number`: `QUANTITY`, 64 Bits

#### L1BlockRef

- `hash`: `DATA`, 32 Bytes
- `number`: `QUANTITY`, 64 Bits
- `parentHash`: `DATA`, 32 Bytes
- `timestamp`: `QUANTITY`, 64 Bits

#### L2BlockRef

- `hash`: `DATA`, 32 Bytes
- `number`: `QUANTITY`, 64 Bits
- `parentHash`: `DATA`, 32 Bytes
- `timestamp`: `QUANTITY`, 64 Bits
- `l1origin`: `BlockID`
- `sequenceNumber`: `QUANTITY`, 64 Bits - distance to first block of epoch

#### SyncStatus

Represents a snapshot of the rollup driver.

- `current_l1`: `Object` - instance of [`L1BlockRef`](#l1blockref).
- `current_l1_finalized`: `Object` - instance of [`L1BlockRef`](#l1blockref).
- `head_l1`: `Object` - instance of [`L1BlockRef`](#l1blockref).
- `safe_l1`: `Object` - instance of [`L1BlockRef`](#l1blockref).
- `finalized_l1`: `Object` - instance of [`L1BlockRef`](#l1blockref).
- `unsafe_l2`: `Object` - instance of [`L2BlockRef`](#l2blockref).
- `safe_l2`: `Object` - instance of [`L2BlockRef`](#l2blockref).
- `finalized_l2`: `Object` - instance of [`L2BlockRef`](#l2blockref).
- `pending_safe_l2`: `Object` - instance of [`L2BlockRef`](#l2blockref).
- `queued_unsafe_l2`: `Object` - instance of [`L2BlockRef`](#l2blockref).

### Output Method API

The input and return types here are as defined by the [engine API specs][engine-structures].

[engine-structures]: https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md#structures

- method: `optimism_outputAtBlock`
- params:
  1. `blockNumber`: `QUANTITY`, 64 bits - L2 integer block number </br>
        OR `String` - one of `"safe"`, `"latest"`, or `"pending"`.
- returns:
  1. `version`: `DATA`, 32 Bytes - the output root version number, beginning with 0.
  2. `l2OutputRoot`: `DATA`, 32 Bytes - the output root.

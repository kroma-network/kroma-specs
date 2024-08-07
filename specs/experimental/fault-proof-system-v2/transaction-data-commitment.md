# Transaction Data Commitment

<!-- All glossary references in this file. -->

[g-zk-fault-proof]: ../../glossary.md#zk-fault-proof
[g-l1-attr-deposit]: ../../glossary.md#l1-attributes-deposited-transaction
[g-user-deposited]: ../../glossary.md#user-deposited-transaction
[g-system-config]: ../../glossary.md#system-configuration
[g-l1-attr-predeploy]: ../../glossary.md#l1-attributes-predeployed-contract
[g-state]: ../../glossary.md#state
[g-avail-provider]: ../../glossary.md#data-availability-provider
[g-sequencer]: ../../glossary.md#sequencer
[g-batcher-transaction]: ../../glossary.md#batcher-transaction
[g-channel-frame]: ../../glossary.md#channel-frame
[g-channel]: ../../glossary.md#channel
[g-sequencer-batch]: ../../glossary.md#sequencer-batch

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Types of Transactions in Kroma L2 Block](#types-of-transactions-in-kroma-l2-block)
- [Deposited Transactions](#deposited-transactions)
  - [L1 Attributes Deposited Transactions](#l1-attributes-deposited-transactions)
  - [User-Deposited Transactions](#user-deposited-transactions)
- [L2 Transactions](#l2-transactions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

This document outlines the process of on-chain verification of transaction data in the context of
[ZK fault proof][g-zk-fault-proof]. It's essential for asserters or challengers to prove that their ZK fault proof was
generated by correctly executing the relevant transactions of a target L2 block in the zkEVM.

## Types of Transactions in Kroma L2 Block

The Kroma L2 block comprises three types of transactions:

- [L1 Attributes Deposited Transactions][g-l1-attr-deposit]
- [User-Deposited Transactions][g-user-deposited]
- L2 Transactions

The sections below detail how commitments for these transactions are stored on-chain and utilized for ZK proof
verification.

## Deposited Transactions

### L1 Attributes Deposited Transactions

L1 attributes deposited transactions store information from the L1 reference block and
[L2 system configuration][g-system-config] in the [L1Block][g-l1-attr-predeploy] contract. Automatically generated as
the first transaction of every new L2 block, they change L2 [state][g-state] and hence must be included in ZK proof
verification. However, they are not currently part of [batch data][g-sequencer-batch], making on-chain verification
challenging. Future updates will address this with specific verification instructions.

TBD

### User-Deposited Transactions

User-deposited transactions are transactions that are deposited from L1 to L2 through the `KromaPortal`. To verify
user-deposited transactions on-chain, an accumulated hash is computed and stored as a commitment each time a
user-deposited transaction occurs within the same L1 block. It is calculated and stored as follows:

```solidity
/**
 * @notice A mapping of L1 block number to the accumulated hash of the user-deposited transaction messages
 *         included in the L1 block.
 */
mapping(uint256 => bytes32) public userDepositedTxAccHash;

/**
 * @notice Store the cumulatively hashed message of user-deposited transactions by hashing it with the previously
 *         accumulated message hash corresponding to the L1 block number.
 *
 * @param userDepositedTxMsgHash The hash of new user-deposited transaction message.
 */
function addUserDepositedTxToAccHash(bytes32 userDepositedTxMsgHash) internal onlyKromaPortal {
    userDepositedTxAccHash[block.number] = keccak256(
        abi.encodePacked(userDepositedTxAccHash[block.number], userDepositedTxMsgHash)
    );
}
```

The accumulated hash containing all user-deposited transactions in the target block is retrievable on-chain, and can be
used for [ZK proof verification](./challenge.md#confirm-by-zk-proof) to verify the correct execution of user-deposited
transactions.

```solidity
/**
 * @notice Returns an accumulated hash of the user-deposited transactions included in the given L1 block.
 *         Returns zero if the accumulated hash is not found corresponding to the given L1 block number.
 *
 * @param l1BlockRefNum The L1 block number including the user-deposited transactions.
 *
 * @return The accumulated hash of the user-deposited transactions included in the given L1 block.
 */
function getUserDepositedTxAccHash(uint256 l1BlockRefNum) external view returns (bytes32);
```

## L2 Transactions

L2 transactions are submitted by a [sequencer][g-sequencer] to [data availability provider][g-avail-provider]. With
[Proto-Danksharding](https://www.eip4844.com/), the sequencer can roll up batch data to the data availability provider
in a more cost-effective manner while storing the commitments(`versionedHash`) of the data on-chain. These commitments
are verified by beacon nodes when the sequencer submits the data, and can be used for
[ZK proof verification](./challenge.md#confirm-by-zk-proof).

A [batcher transaction][g-batcher-transaction] contains one or more [channel frames][g-channel-frame], which are chunks
of data belonging to a [channel][g-channel]. A channel is a sequence of [sequencer batches][g-sequencer-batch]
compressed together. Note that a single batcher transaction can carry frames from multiple channels, which can be
multiple blobs. All blobs in the batcher transaction containing the target block data should be checked to verify that
the correct transaction data is used to generate the ZK proof. Therefore, the sequencer needs to store `versionedHashes`
for all blobs in the batcher transaction.

```solidity
/**
 * @notice A mapping indicating whether a versioned hash is stored or not.
 */
mapping(bytes32 => bool) public BIBVersionedHashes;

/**
 * @notice Store the versioned hashes of the L2 batch blobs. This function should always be called when the
 *         sequencer submits batch data.
 */
function addBIBVersionedHashes() external onlySequencer {
    for (uint256 i = 0; i < tx.blob_versioned_hashes.length; i++) {
        BIBVersionedHashes[tx.blob_versioned_hashes[i]] = true;
    }
}
```

Whether `versionedHash` of the batcher transaction containing the target block data is stored on-chain or not can be
checked as follows.

```solidity
/**
 * @notice Returns the existence of the given versioned hash.
 *
 * @param versionedHash The versioned hash.
 *
 * @return The existence of the given versioned hash.
 */
function isBIBVersionedHash(bytes32 versionedHash) external view returns (bool);
```

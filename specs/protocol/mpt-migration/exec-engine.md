# L2 Execution Engine

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Timestamp Activation](#timestamp-activation)
- [EVM Changes](#evm-changes)
  - [Transition from ZK Trie to MPT](#transition-from-zk-trie-to-mpt)
  - [`SELFDESTRUCT` opcode](#selfdestruct-opcode)
  - [`isSystemTransaction` boolean at transaction struct](#issystemtransaction-boolean-at-transaction-struct)
- [Fee Distribution Process](#fee-distribution-process)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

[g-zktrie]: ../../glossary.md#zk-trie
[g-mpt]: ../../glossary.md#merkle-patricia-trie
[validator-system]: ../validator-v2/overview.md

## Overview

This upgrade changes the underlying trie structure of the execution client from the [ZK Trie][g-zktrie] to the
[Merkle Patricia Trie][g-mpt]. There are also subsequent changes to the EVM to enhance the compatibility with
Vanilla OP Stack.

## Timestamp Activation

Kroma MPT Migration upgrade, like other network upgrades, is activated at a timestamp. Changes to the L2 Block execution
rules are applied when the `L2 Timestamp >= activation time`.

## EVM Changes

### Transition from ZK Trie to MPT

The underlying trie structure of the execution client is changed from the ZK Trie to the Merkle Patricia Trie. This
enhances the performance of the execution client, and reduces the operational cost of maintaining zkEVM circuits by
enabling the proving scheme transition to zkVM.

### `SELFDESTRUCT` opcode

The `SELFDESTRUCT` opcode was regarded as invalid opcode, since zkEVM proving system used in ZK fault proof cannot prove
the self-destruct operation. However, with the transition to zkVM, the `SELFDESTRUCT` opcode is now enabled, following
rules that are defined at [EIP-6780].

[EIP-6780]: https://eips.ethereum.org/EIPS/eip-6780

### `isSystemTransaction` boolean at transaction struct

In the previous version of the protocol, the `isSystemTransaction` boolean was removed. To enhance the compatibility
with vanilla OP Stack, the `isSystemTransaction` boolean is re-enabled.

However, since `isSystemTransaction` was disabled as in the [Regolith upgrade](../regolith/overview.md), system
transactions remain to use the same gas accounting rules as regular deposits.

## Fee Distribution Process

The fee distribution process is updated to be compatible with OP Stack. Previously, the fee distribution process was
defined to support robustness of validator system.

| Vault                  | Address                                      | Amount                                                                |
|------------------------|----------------------------------------------|-----------------------------------------------------------------------|
| `ProtocolVault`        | `0x4200000000000000000000000000000000000006` | `(base_fee + priority_fee) * (10000 - validatorRewardScalar) / 10000` |
| `L1FeeVault`           | `0x4200000000000000000000000000000000000007` | `l1_cost`                                                             |
| `ValidatorRewardVault` | `0x4200000000000000000000000000000000000008` | `(base_fee + priority_fee) * validatorRewardScalar / 10000`           |

However, with transition to [KRO-based validator system][validator-system], validators are now getting KRO rewards,
and `validatorRewardScalar` was set to 0. After Kroma MPT Migration upgrade, the fee distribution process is updated
as follows:

| Vault               | Address                                      | Amount         |
|---------------------|----------------------------------------------|----------------|
| `SequencerFeeVault` | `0x4200000000000000000000000000000000000011` | `priority_fee` |
| `BaseFeeVault`      | `0x4200000000000000000000000000000000000019` | `base_fee`     |
| `L1FeeVault`        | `0x420000000000000000000000000000000000001a` | `l1_cost`      |

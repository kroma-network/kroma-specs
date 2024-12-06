# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Changes on Predeploys](#changes-on-predeploys)
  - [L1Block](#l1block)
  - [L1FeeVault](#l1feevault)
  - [ProtocolVault](#protocolvault)
  - [SequencerFeeVault](#sequencerfeevault)
  - [BaseFeeVault](#basefeevault)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Changes on Predeploys

### L1Block

Following the Kroma MPT upgrade, the calldata to set values in the `L1Block` predeploy is updated to
exclude the `validatorRewardScalar`. Now the calldata is as follows:

```text
baseFeeScalar ++ blobBaseFeeScalar ++ sequenceNumber ++ timestamp ++ blockNumber ++ baseFee ++ blobBaseFee ++ blockHash ++ batcherHash
```

Also, the address of `L1Block` is changed from `0x4200000000000000000000000000000000000002` to
`0x4200000000000000000000000000000000000015`. The original address is now deprecated and not used after the Kroma MPT
hard fork.

### L1FeeVault

The address of `L1FeeVault` is changed from `0x4200000000000000000000000000000000000007` to
`0x420000000000000000000000000000000000001a`. The original address is now deprecated and not used after the Kroma MPT
hard fork.

### ProtocolVault

The `ProtocolVault` is now deprecated and not used after the Kroma MPT hard fork.

### SequencerFeeVault

The new predeploy `SequencerFeeVault` is introduced with the address `0x4200000000000000000000000000000000000011` after
the Kroma MPT hard fork.

### BaseFeeVault

The new predeploy `BaseFeeVault` is introduced with the address `0x4200000000000000000000000000000000000019` after
the Kroma MPT hard fork.

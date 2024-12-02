# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [ProxyAdmin](#proxyadmin)
- [WETH9](#weth9)
- [L2ToL1MessagePasser](#l2tol1messagepasser)
- [L2CrossDomainMessenger](#l2crossdomainmessenger)
- [GasPriceOracle](#gaspriceoracle)
- [ProtocolVault](#protocolvault)
- [ValidatorRewardVault](#validatorrewardvault)
- [L2StandardBridge](#l2standardbridge)
- [KromaMintableERC20Factory](#kromamintableerc20factory)
- [KromaMintableERC721Factory](#kromamintableerc721factory)
- [SequencerFeeVault](#sequencerfeevault)
- [L1Block](#l1block)
- [BaseFeeVault](#basefeevault)
- [L1FeeVault](#l1feevault)
- [Beacon Block Root](#beacon-block-root)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->
[g-predeployed-contract-predeploy]: ../glossary.md#predeployed-contract-predeploy

## Overview

[Predeployed smart contracts][g-predeployed-contract-predeploy] exist on Kroma
at predetermined addresses in the genesis state. They are  similar to precompiles but instead run
directly in the EVM instead of running  native code outside of the EVM.

Predeploys are used instead of precompiles to make it easier for multiclient
implementations as well as allowing for more integration with hardhat/foundry
network forking.

Predeploy addresses exist in 1 byte namespace `0x42000000000000000000000000000000000000xx`.
Proxies are set at each possible predeploy address except for the `ProxyAdmin` and the `WETH9`.

| Name                       | Address                                    | Introduced | Deprecated | Proxied |
|----------------------------|--------------------------------------------|------------|------------|---------|
| ProxyAdmin                 | 0x4200000000000000000000000000000000000000 | Legacy     | No         | No      |
| WETH9                      | 0x4200000000000000000000000000000000000001 | Legacy     | No         | No      |
| L1Block (*deprecated*)     | 0x4200000000000000000000000000000000000002 | Bedrock    | Yes        | Yes     |
| L2ToL1MessagePasser        | 0x4200000000000000000000000000000000000003 | Legacy     | No         | Yes     |
| L2CrossDomainMessenger     | 0x4200000000000000000000000000000000000004 | Legacy     | No         | Yes     |
| GasPriceOracle             | 0x4200000000000000000000000000000000000005 | Legacy     | No         | Yes     |
| ProtocolVault              | 0x4200000000000000000000000000000000000006 | Bedrock    | Yes        | Yes     |
| L1FeeVault (*deprecated*)  | 0x4200000000000000000000000000000000000007 | Bedrock    | Yes        | Yes     |
| ValidatorRewardVault       | 0x4200000000000000000000000000000000000008 | Bedrock    | Yes        | Yes     |
| L2StandardBridge           | 0x4200000000000000000000000000000000000009 | Bedrock    | No         | Yes     |
| L2ERC721Bridge             | 0x420000000000000000000000000000000000000A | Legacy     | No         | Yes     |
| KromaMintableERC20Factory  | 0x420000000000000000000000000000000000000B | Legacy     | No         | Yes     |
| KromaMintableERC721Factory | 0x420000000000000000000000000000000000000C | Bedrock    | No         | Yes     |
| SequencerFeeVault          | 0x4200000000000000000000000000000000000011 | Legacy     | No         | Yes     |
| L1Block                    | 0x4200000000000000000000000000000000000015 | Bedrock    | No         | Yes     |
| BaseFeeVault               | 0x4200000000000000000000000000000000000019 | Yes        | No         | Yes     |
| L1FeeVault                 | 0x420000000000000000000000000000000000001a | Bedrock    | No         | Yes     |
| BeaconBlockRoot            | 0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02 | Ecotone    | No         | No      |

## ProxyAdmin

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/universal/ProxyAdmin.sol)

Address: `0x4200000000000000000000000000000000000000`

The `ProxyAdmin` is the owner of all of the proxy contracts set at the
predeploys. It is itself behind a proxy. The owner of the `ProxyAdmin` will
have the ability to upgrade any of the other predeploy contracts.

## WETH9

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/vendor/WETH9.sol)

Address: `0x4200000000000000000000000000000000000001`

`WETH9` is the standard implementation of Wrapped Ether on Kroma. It is a
commonly used contract and is placed as a predeploy so that it is at a
deterministic address across Kroma based networks.

## L2ToL1MessagePasser

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/L2ToL1MessagePasser.sol)

Address: `0x4200000000000000000000000000000000000003`

The `L2ToL1MessagePasser` stores commitments to withdrawal transactions.
When a user is submitting the withdrawing transaction on L1, they provide a
proof that the transaction that they withdrew on L2 is in the `sentMessages`
mapping of this contract.

Any withdrawn ETH accumulates into this contract on L2 and can be
permissionlessly removed from the L2 supply by calling the `burn()` function.

## L2CrossDomainMessenger

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/L2CrossDomainMessenger.sol)

Address: `0x4200000000000000000000000000000000000004`

The `L2CrossDomainMessenger` gives a higher level API for sending cross domain
messages compared to directly calling the `L2ToL1MessagePasser`.
It maintains a mapping of L1 messages that have been relayed to L2
to prevent replay attacks and also allows for replayability if the L1 to L2
transaction reverts on L2.

Any calls to the `L1CrossDomainMessenger` on L1 are serialized such that they
go through the `L2CrossDomainMessenger` on L2.

The `relayMessage` function executes a transaction from the remote domain while
the `sendMessage` function sends a transaction to be executed on the remote
domain through the remote domain's `relayMessage` function.

## GasPriceOracle

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/GasPriceOracle.sol)

Address: `0x4200000000000000000000000000000000000005`

The `GasPriceOracle` provides an API for offchain gas estimation. The
function `getL1Fee(bytes)` accepts an unsigned RLP transaction and will return
the L1 portion of the fee. This fee pays for using L1 as a data availability
layer and should be added to the L2 portion of the fee, which pays for
execution, to compute the total transaction fee.

The values used to compute the L2 portion of the fee are:

- scalar
- overhead
- decimals

These values are managed by the `SystemConfig` contract on L2. The `scalar` and
`overhead` values are sent to the `L1Block` contract each block and the `decimals`
value is hardcoded to 6.

## ProtocolVault

**NOTE**: This contract is deprecated and no longer in use.

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/ProtocolVault.sol)

Address: `0x4200000000000000000000000000000000000006`

The `ProtocolVault` predeploy accumulates transaction fees to fund network operation.
Once the contract has received a certain amount of fees, the ETH can be
withdrawn to an immutable address on L1.

## ValidatorRewardVault

**NOTE**: This contract is deprecated and no longer in use.

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/ValidatorRewardVault.sol)

Address: `0x4200000000000000000000000000000000000008`

The `ValidatorRewardVault` accumulates transaction fees and pays rewards to validators in
[ETH-based Validator System](../deprecated/validator-v1/validator-pool.md).
When enough fees accumulate in this account, they can be withdrawn to an immutable L1 address.

To change the L1 address that fees are withdrawn to, the contract must be
upgraded by changing its proxy's implementation key.

## L2StandardBridge

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/L2StandardBridge.sol)

Address: `0x4200000000000000000000000000000000000009`

The `L2StandardBridge` is a higher level API built on top of the
`L2CrossDomainMessenger` that gives a standard interface for sending ETH or
ERC20 tokens across domains.

To deposit a token from L1 to L2, the `L1StandardBridge` locks the token and
sends a cross domain message to the `L2StandardBridge` which then mints the
token to the specified account.

To withdraw a token from L2 to L1, the user will burn the token on L2 and the
`L2StandardBridge` will send a message to the `L1StandardBridge` which will
unlock the underlying token and transfer it to the specified account.

The `KromaMintableERC20Factory` can be used to create an ERC20 token contract
on a remote domain that maps to an ERC20 token contract on the local domain
where tokens can be deposited to the remote domain. It deploys an
`KromaMintableERC20` which has the interface that works with the
`StandardBridge`.

This contract can also be deployed on L1 to allow for L2 native tokens to be
withdrawn to L1.

## KromaMintableERC20Factory

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/universal/KromaMintableERC20Factory.sol)

Address: `0x420000000000000000000000000000000000000B`

The `KromaMintableERC20Factory` is responsible for creating ERC20 contracts on L2 that can be
used for depositing native L1 tokens into. These ERC20 contracts can be created permisionlessly
and implement the interface required by the `StandardBridge` to just work with deposits and withdrawals.

Each ERC20 contract that is created by the `KromaMintableERC20Factory` allows for the `L2StandardBridge` to mint
and burn tokens, depending on if the user is depositing from L1 to L2 or withdrawing from L2 to L1.

## KromaMintableERC721Factory

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/universal/KromaMintableERC721Factory.sol)

Address: `0x420000000000000000000000000000000000000C`

The `KromaMintableERC721Factory` is responsible for creating ERC721 contracts on L2 that can be used for
depositing native L1 NFTs into.

## SequencerFeeVault

[Implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/SequencerFeeVault.sol)

Address: `0x4200000000000000000000000000000000000011`

The `SequencerFeeVault` accumulates any transaction priority fee and is the value of
`block.coinbase`.
When enough fees accumulate in this account, they can be withdrawn to an immutable L1 address.

To change the L1 address that fees are withdrawn to, the contract must be
upgraded by changing its proxy's implementation key.

## L1Block

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/L1Block.sol)

Address: `0x4200000000000000000000000000000000000015`

[l1-block-predeploy]: ../glossary.md#l1-attributes-predeployed-contract

The [L1Block][l1-block-predeploy] is responsible for maintaining L1 context in L2.
This allows for L1 state to be accessed in L2.

## BaseFeeVault

[Implementation](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L2/BaseFeeVault.sol)

Address: `0x4200000000000000000000000000000000000019`

The `BaseFeeVault` predeploy receives the base fees on L2. The base fee is not
burnt on L2 like it is on L1. Once the contract has received a certain amount
of fees, the ETH can be withdrawn to an immutable address on
L1.

## L1FeeVault

[Implementation](https://github.com/kroma-network/kroma/blob/main/packages/contracts/contracts/L2/L1FeeVault.sol)

Address: `0x420000000000000000000000000000000000001a`

The `L1FeeVault` predeploy receives the L1 portion of the transaction fees.
Once the contract has received a certain amount of fees, the ETH can be
withdrawn to an immutable address on L1.

## Beacon Block Root

Address: `0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02`

The `BeaconBlockRoot` predeploy provides access to the L1 beacon block roots. This was added during the
Ecotone network upgrade and is specified in [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788).

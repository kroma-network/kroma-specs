# Asset Manager

<!-- All glossary references in this file. -->

[g-l2-output]: ../../glossary.md#l2-output-root
[g-validator]: ../../glossary.md#validator
[g-validator-manager-contract]: ../../glossary.md#validator-manager-contract

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Composition of Asset Manager](#composition-of-asset-manager)
- [Reward Management](#reward-management)
  - [Base Rewards](#base-rewards)
  - [Boosted Rewards](#boosted-rewards)
- [Validator Asset Management](#validator-asset-management)
- [Undelegation Delay of 7 days from the Delegation](#undelegation-delay-of-7-days-from-the-delegation)
- [Difference from ERC-4626](#difference-from-erc-4626)
  - [Non-transferable Shares](#non-transferable-shares)
  - [Global Vault Structure](#global-vault-structure)
- [Security Considerations](#security-considerations)
  - [Virtual Offset](#virtual-offset)
  - [Virtual Share](#virtual-share)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Asset Manager oversees the delegation and undelegation of KRO tokens and Kroma Guardian House (KGH) NFTs.
It distributes rewards based on [output root][g-l2-output] submissions and manages slashing penalties. KRO and KGH
delegators delegate their assets into the system and receive rewards upon undelegation. The Asset Manager supervises the
assets of all [validators][g-validator] and delegators within the Kroma validator system.

## Composition of Asset Manager

Asset Manager incorporates a `Vault` component, inspired by the
[ERC-4626 standard](https://eips.ethereum.org/EIPS/eip-4626) and
[Synthetix StakingRewards contract](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol).
Its responsibilities include:

1. Management of delegated KRO and KGH

    - Adopts the structure of ERC-4626 to issue shares when KROs are delegated, and that of Synthetix StakingRewards
      contract to manage boosted rewards when KGHs are delegated.

2. Slashing and Reward Management

    - Manages rewards for delegators based on the output root submission and slashing of validator assets when
      challenges occur.

Each validator registered in [Validator Manager contract][g-validator-manager-contract] has a corresponding `Vault`
struct. The `Vault` struct is defined as follows:

```solidity
struct Asset {
    uint128 validatorKro;
    uint128 validatorKroBonded;
    uint128 totalKro;
    uint128 totalKroShares;
    uint128 totalKgh;
    uint128 rewardPerKghStored;
}

struct Vault {
    address withdrawAccount;
    uint128 lastDepositedAt;
    Asset asset;
    mapping(address => KroDelegator) kroDelegators;
    mapping(address => KghDelegator) kghDelegators;
}
```

The `Asset` struct contain global information such as the total number of KROs in the `Vault`. In
contrast, the `KroDelegator` and `KghDelegator` structs store individual information about each delegator,
such as the number of shares he hold in the `Vault`. `withdrawAccount` and `lastDepositedAt` are the information
specific to the validator.

The `Vault` defines assets and shares for KRO and streaming-based reward for KGH, allowing for distinct reward
distribution mechanisms for KRO delegators and KGH delegators.

## Reward Management

The base reward is given by increasing the price of `kroShare`, which is given to the delegator when he delegates KRO.
On the other side, the boosted reward is given by increasing `rewardPerKghStored` at the `Asset` struct. The share
and reward information for each delegator is respectively stored in the `KroDelegator` and `KghDelegator` struct, which
is separately allocated for every delegators.

```solidity
struct KroDelegator {
    uint128 shares;
    uint128 lastDelegatedAt;
}

struct KghDelegator {
    uint128 rewardPerKghPaid;
    uint128 kghNum;
    mapping(uint256 => uint128) delegatedAt;
}
```

### Base Rewards

If a delegator delegates KRO, they receive `kroShare` based on the current state of the `Vault`. The number of shares
issued is calculated similarly to ERC-4626, as follows:

$$kroShare = kroAsset \times \frac{totalKroShares+10^{offset}}{totalKroAssets+1}$$

If a delegator wants to undelegate all their assets by burning `kroShare`, the amount of KRO they receive is calculated
as follows:

$$kroAsset = kroShare \times \frac{totalKroAssets+1}{totalKroShares+10^{offset}}$$

Information related to the assets and shares of each validator is stored in the following `Asset` struct. Note that the
base reward is added to the `totalKro` field, increasing the share price.

### Boosted Rewards

Boosted rewards are managed by calculating the expected rewards per KGH each time a submitted output is finalized and
rewards are distributed. These are stored within the `Vault` as `rewardPerKghStored`.

```solidity
    vault.asset.rewardPerKghStored += boostedReward / vault.asset.totalKgh
```

The `boostedReward` refers to the total boosted reward from KGH delegations for the validator, and
`vault.asset.totalKgh` represents the total number of KGH NFTs delegated to the validator.

When KGH delegators claim their boosted rewards, the rewards they will receive are calculated as follows and directly
transferred to the delegator's account.

```solidity
    uint128 totalBoostedReward = kghDelegator.kghNum * (vault.asset.rewardPerKghStored - kghDelegator.rewardPerKghPaid);
    kghDelegator.rewardPerKghPaid = vault.asset.rewardPerKghStored;
    ASSET_TOKEN.safeTransfer(delegator, totalBoostedReward);
```

`kghDelegator.rewardPerKghPaid` stores the rewards per KGH that the current delegator has received. This ensures that
the rewards are correctly distributed to KGH delegators when new delegations or undelegations occur.

## Validator Asset Management

Validator assets are also managed inside the `Asset` struct. The `validatorKro` field stores the amount of KRO deposited
by the validator, while the `validatorKroBonded` field stores the amount of KRO that is bonded for the output
submission or creation of a challenge.

The deposited validator KROs contribute to the probability of the validator being selected as a priority validator,
and also validators are eligible for the portion of the base reward in proportion to the amount of KRO they deposited.

To withdraw the deposited KRO, the validator should call withdraw function with its withdrawAccount, which is a
dedicated account for the validator to withdraw the KRO. Separating the withdraw account from the validator's account
can be thought as adding an extra security layer to their account, since he can safely withdraw his KRO even if
his validator account is compromised. Please note that the following 2 things to prevent irreversible situations:

- Validators cannot change the address of the withdraw account, so they **MUST** set the withdraw account correctly,
with the wallet address whose private key is possessed by the validator.
- If withdraw account is compromised, the validator will not be able to withdraw the KRO, so validators are **highly
recommended to store the private key of the withdraw account in a safe place**.

## Undelegation Delay of 7 days from the Delegation

To prevent abuse such as attempting to receive unintended and unjustified rewards, all undelegations are subject to a
one-week delay. Without this 7-day delay, the following abuses could occur:

> Exploiting the fact that output finalization takes one week, a delegator could delegate a large amount of KRO right
before output finalization, and then undelegate it immediately after finalization. This way, the delegator could take
the rewards with almost no risk.

To address this edge case, funds are not be able to be undelegated for a week from the time of the delegation. This is
managed through the `lastDelegatedAt` field in the `KroDelegator` struct and `delegatedAt` mapping `KghDelegator`
struct respectively.

## Difference from ERC-4626

### Non-transferable Shares

Unlike ERC-4626, `kroShares` in `Vault` are designed to be non-transferable to any accounts except for the delegator
and Asset Manager contract. This is implemented to prevent the following edge case:

- Transferring shares related to the validator's `minRegisterAmount` (minimum amount of KRO to be a validator) to
  another address.

This case can lead to inconsistent state management of the `Vault`, which could result in unexpected behavior.

### Global Vault Structure

In ERC-4626, each contract typically has a single ERC-20 share. However, `Vault` in Asset Manager manages
deposit, delegation, and rewards for all validators within a single contract.
Therefore, `Vault` operates a mapping with validator addresses as keys and stores the state of assets, shares,
and rewards for each validator's vault within the mapping.

## Security Considerations

To prevent a [vault inflation attack](https://docs.openzeppelin.com/contracts/4.x/erc4626#inflation-attack), two
additional elements are introduced when calculating the amount of shares corresponding to delegated and undelegated
assets.

### Virtual Offset

The exchange rate between shares and assets is artificially increased to prevent attacks due to rounding issues.
This means increasing the decimal places for shares so that each asset maps to $10^{\delta}$ shares.

### Virtual Share

To maintain the exchange rate defined by the virtual offset even in an empty vault where no assets have been delegated
yet, virtual assets and shares are included in the share calculation logic as follows.

$$asset = share \times \frac{totalKroAssets+1}{totalKroShares+10^{offset}}$$

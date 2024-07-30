# KRO-based Validator System

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Background](#background)
- [Overview](#overview)
  - [Participants](#participants)
    - [Validators](#validators)
    - [KRO Delegators](#kro-delegators)
    - [KGH Delegators](#kgh-delegators)
  - [Priority Validator Selection](#priority-validator-selection)
  - [Reward Distribution](#reward-distribution)
- [Contracts](#contracts)
- [Summary of Definitions](#summary-of-definitions)
  - [Constants](#constants)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->

[g-l2-output]: ../../glossary.md#l2-output-root
[g-zk-fault-proof]: ../../glossary.md#zk-fault-proof
[g-validator-pool-contract]: ../../glossary.md#validator-pool-contract
[g-validator-manager-contract]: ../../glossary.md#validator-manager-contract
[g-asset-manager-contract]: ../../glossary.md#asset-manager-contract
[g-validator]: ../../glossary.md#validator
[g-output-reward]: ../../glossary.md#output-reward
[g-validator-reward]: ../../glossary.md#validator-reward
[g-base-reward]: ../../glossary.md#base-reward
[g-boosted-reward]: ../../glossary.md#boosted-reward
[g-priority-round]: ../../glossary.md#priority-round

## Background

The [previous permissionless validator system](../validator-v1/validator-pool.md) of Kroma requires validators to bond
`REQUIRED_BOND_AMOUNT` of ETH each time the validator submits an [L2 output root][g-l2-output]. While successfully
involving over 360 validators (as of 6/3/24), there are a few drawbacks in the current system that need to be addressed:

- **Multiple Accounts on a Single Validator**: Since the probability of being selected as a priority validator is
  calculated to be the same if only `REQUIRED_BOND_AMOUNT` of ETH is deposited, a single validator can increase the
  probability by splitting assets between many accounts. This behavior can have a negative impact on chain security and
  should be improved.
- **Poor Accessibility to Participation**: Despite having one of the largest validator pools among current L2 networks,
  technical barriers limit participation to those who can run validators, excluding many people who would like to
  contribute to the network security and earn incentives.

These limitations are addressed by introducing Delegated Proof of Stake (DPoS) system to the new validator system. The
new validator system will fortify the security of the entire network and lower the barrier of participation,
incorporating delegation system based on Kroma's governance token (KRO) and Kroma Guardian House (KGH) NFT.

## Overview

**Validator System V2** is a new network security model of Kroma, which introduces KRO tokenomics and delegation to the
validator system. It is compatible with current [ZK fault proof][g-zk-fault-proof] and
[challenge system](../../fault-proof/challenge.md), while introducing two new contracts:
[**Validator Manager**][g-validator-manager-contract] and [**Asset Manager** contract][g-asset-manager-contract]. These
contracts replace the existing [Validator Pool contract][g-validator-pool-contract], handling jobs such as validator
management, delegation, next priority validator selection, [output reward][g-output-reward] distribution, and challenge
slashing.

### Participants

There are three types of participants in the Validator System V2: Validators, KRO delegators, and KGH delegators.

#### Validators

[Validators][g-validator] are the entities who actually run the node and submit the output root, which are the same as
validators in the previous validator system. Validators must deposit their KRO tokens to be eligible to submit output,
and their chances of submitting output increase proportionally to the amount of KRO they deposit. If the submitted
output is finalized, they will receive KRO as a [reward][g-validator-reward], while if the output is challenged and
lost, a portion of the deposited KRO and the output submission reward will be transferred to the challenge winner.

#### KRO Delegators

KRO delegators delegate their KRO to validators and increase the probability of the validators being selected as a
priority validator for output submission. As compensation for the delegation, KRO delegators receive a portion of the
[base reward][g-base-reward] which the validators earn for submitting output, proportional to the number of delegated
KRO.

#### KGH Delegators

KGH delegators delegate KGHs to validators and boost their output submission rewards. In return for delegation,
they receive a share in the [boosted reward][g-boosted-reward].

### Priority Validator Selection

The probability of a validator being selected as a validator who has priority for the next
[priority round][g-priority-round] is proportional to the amount of KRO delegated to the validator. The amount of
delegated KRO includes both the amount of KRO deposited by themselves and the amount of KRO delegated by KRO delegators.

### Reward Distribution

The reward for submitting an output consists of [base reward][g-base-reward] and [boosted reward][g-boosted-reward].
Base reward is divided between the validator and KRO delegators according to the commission rate set by the validator.
At this point, the validator also share the reward distributed to the KRO delegation based on the
amount of KRO they deposited. Boosted reward is shared by the validator and KGH delegators in proportion to the
commission rate.

## Contracts

- [Validator Manager](validator-manager.md)
- [Asset Manager](asset-manager.md)

## Summary of Definitions

### Constants

| Name                   | Value                        | Unit |
|------------------------|------------------------------|------|
| `REQUIRED_BOND_AMOUNT` | 200000000000000000 (0.2 ETH) | wei  |

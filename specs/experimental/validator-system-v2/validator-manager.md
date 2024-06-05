
# Validator Manager

<!-- All glossary references in this file. -->

[g-validator]: ../../glossary.md#validator
[g-l2-output]: ../../glossary.md#l2-output-root
[output-oracle]: ../../glossary.md#l2-output-oracle-contract
[colosseum-contract]: ../../glossary.md#colosseum-contract
[g-validator-reward]: ../../glossary.md#validator-reward

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Validator Registration](#validator-registration)
- [Validator Management](#validator-management)
  - [Validator Tree](#validator-tree)
  - [Status of Validator](#status-of-validator)
  - [Jail](#jail)
- [Entry Point](#entry-point)
  - [Slashing](#slashing)
  - [Reward Distribution](#reward-distribution)
- [Summary of Definitions](#summary-of-definitions)
  - [Constants](#constants)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Validator Manager manages the set of [validators][g-validator] and selects the next priority validator with the
priority to submit the [output root][g-l2-output]. It is also the entry point for other contracts, such as the
[L2 Output Oracle][output-oracle] and the [Colosseum][colosseum-contract], which distribute
[output rewards][g-validator-reward] and slash the challenge losers. It makes successive calls to the
[Asset Manager contract](./asset-manager.md) to apply changes to the validators' assets.

## Validator Registration

To become a validator, the first step is to register to the Validator Manager with at least `MIN_REGISTER_AMOUNT` of
tokens (KRO). Upon registration, they initiate their [`Vault`](./asset-manager.md#composition-of-asset-manager) and set
commission rate and maximum change rate of the commission rate. The commission rate is the percentage that the validator
takes from the output reward, and the maximum change rate is the maximum range of the commission rate that the validator
can change at one time. If `COMMISSION_RATE_MIN_CHANGE_SECONDS` has not elapsed since the last time the validator
changed the commission rate, the validator cannot change it again. These information and safeguards can help delegators
choose which validators to delegate to.

The `Validator` struct that represents the information of the validator is defined as follows:

```solidity
struct Validator {
    bool isInitiated;
    uint8 noSubmissionCount;
    uint8 commissionRate;
    uint8 commissionMaxChangeRate;
    uint128 commissionRateChangedAt;
}
```

After registration, validators can be delegated tokens and KGH NFTs. When the sum of delegated tokens exceeds
`MIN_ACTIVATE_AMOUNT`, validators can activate its status and become eligible to submit output. When calculating the sum
of delegated tokens, tokens contained in KGH are excluded as they are not subject to slashing. This means that
validators should only fill up to `MIN_ACTIVATE_AMOUNT` with real tokens, not virtual ones.

## Validator Management

Once initiated, validators are managed using a data structure called validator tree and categorized statuses to select
priority validator and check output submission eligibility. Below are the implementation details for managing
validators.

### Validator Tree

To select next priority validator based on the amount of tokens delegated to the validator, validators are managed using
a [self-balancing binary search tree structure][self-balancing-binary-search-tree]. When a validator invokes activate,
it is inserted into the tree. The interface of the validator tree is shown below:

```solidity
library BalancedWeightTree {
    struct Node {
        address addr;
        uint32 parent;
        uint32 leftChild;
        uint32 rightChild;
        bool isLeftChild;
        uint120 weight;
        uint120 weightSum;
    }

    struct Tree {
        uint32 counter;
        uint32 removed;
        uint32 root;
        mapping(uint32 => Node) nodes;
        mapping(address => uint32) nodeMap;
    }

    function insert(Tree storage _tree, address _addr, uint120 _weight) internal;
    function update(Tree storage _tree, address _addr, uint120 _weight) internal returns (bool);
    function remove(Tree storage _tree, address _addr) internal returns (bool);
    function select(Tree storage _tree, uint120 _weight) internal view returns (address);
}
```

As described in [Priority Validator Selection](./overview.md#priority-validator-selection), a validator's weight is
calculated by summing self-delegated tokens, tokens delegated by KRO delegators, tokens contained in delegated KGHs, and
cumulative reward. It must be updated after each delegation, undelegation, slashing, and reward distribution. The next
priority validator is randomly selected from the validator tree based on their weight.

[self-balancing-binary-search-tree]: https://github.com/yasharpm/Solidity-Weighted-Random-List

### Status of Validator

Based on some conditions of validators, validator statuses are categorized as follows:

| Status       | initiated | activated | `MIN_REGISTER_AMOUNT` | `MIN_ACTIVATE_AMOUNT` |
|--------------|-----------|-----------|-----------------------|-----------------------|
| `NONE`       | X         | X         | X                     | X                     |
| `EXITED`     | O         | O/X       | X                     | O/X                   |
| `REGISTERED` | O         | X         | O                     | X                     |
| `READY`      | O         | X         | O                     | O                     |
| `INACTIVE`   | O         | O         | O                     | X                     |
| `ACTIVE`     | O         | O         | O                     | O                     |

Only validators with an `ACTIVE` status can submit outputs.

### Jail

If a validator has not submitted output when it was selected as a priority validator, the validator's
`noSubmissionCount` is incremented by 1. If the validator submits output the next time it is selected as a priority
validator, `noSubmissionCount` is reset to 0. Otherwise, if the `noSubmissionCount` exceeds `JAIL_THRESHOLD`, the
validator is deemed to have failed to fulfill its obligations. Accordingly, the validator is sent to jail and removed
from the validator tree. After `JAIL_PERIOD` has elapsed, the validator can be unjailed by calling `tryUnjail` function
on its own and activated if the activation conditions are met.

## Entry Point

Validator Manager contract acts as an entry point for external contracts to slash challenge losers or distribute output
rewards. Below is a detailed description of each case.

### Slashing

When the challenge ends, the challenge loser is slashed. To apply the slash to the loser, the Colosseum contract calls
Validator Manager, which calls Asset Manager to calculate the slash amount and apply the change. The loser is sent to
jail and the slash amount is accumulated as a pending challenge reward.

### Reward Distribution

Each time a submitted output is finalized, the reward for submitting the output is distributed to the output submitter.
As mentioned in [Reward distribution](./overview.md#reward-distribution), each reward is calculated as follows:

| Rewards For Validators                                             | Rewards For KRO Delegators          | Rewards For KGH Delegators             |
|--------------------------------------------------------------------|-------------------------------------|----------------------------------------|
| $\verb#BASE_REWARD# \times c_i + \verb#BOOSTED_REWARD# \times c_i$ | $\verb#BASE_REWARD# \times (1-c_i)$ | $\verb#BOOSTED_REWARD# \times (1-c_i)$ |

Here, $c_i$ is the commission rate the validator set during the validator registration process. `BOOSTED_REWARD` is
calculated based on the number of delegated KGH, the exact expression is $$8 \times arctan(0.01 \times G_i)$$, $G_i$
denotes the number of KGH delegated to $i^{th}$ validator. This is designed to have the effect that the boosted reward
is constantly decreasing while the number of KGH increases.

In addition, any pending challenge rewards accumulated as a result of slash are distributed to the output submitter who
is the challenge winner.

## Summary of Definitions

### Constants

| Name                                 | Value | Unit    |
|--------------------------------------|-------|---------|
| `MIN_REGISTER_AMOUNT`                | TBD   | KRO     |
| `MIN_ACTIVATE_AMOUNT`                | TBD   | KRO     |
| `COMMISSION_RATE_MIN_CHANGE_SECONDS` | TBD   | seconds |
| `JAIL_THRESHOLD`                     | TBD   | number  |
| `BASE_REWARD`                        | TBD   | KRO     |

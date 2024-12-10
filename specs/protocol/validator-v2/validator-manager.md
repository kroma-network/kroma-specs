# Validator Manager

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Validator Registration](#validator-registration)
- [Validator Management](#validator-management)
  - [Bond](#bond)
  - [Validator Tree](#validator-tree)
  - [Status of Validator](#status-of-validator)
  - [Jail](#jail)
- [Entry Point](#entry-point)
  - [Slashing](#slashing)
  - [Reward Distribution](#reward-distribution)
- [Summary of Definitions](#summary-of-definitions)
  - [Constants](#constants)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->

[g-validator]: ../../glossary.md#validator
[g-l2-output]: ../../glossary.md#l2-output-root
[output-oracle]: ../../glossary.md#l2-output-oracle-contract
[colosseum-contract]: ../../glossary.md#colosseum-contract
[asset-manager-contract]: ../../glossary.md#asset-manager-contract
[g-output-reward]: ../../glossary.md#output-reward

## Overview

The Validator Manager manages the set of [validators][g-validator] and selects the next priority validator with the
priority to submit the [output root][g-l2-output]. It is also the entry point for other contracts, such as the
[L2 Output Oracle][output-oracle] and the [Colosseum][colosseum-contract], which distribute
[output rewards][g-output-reward] and slash the challenge losers. It makes successive calls to the
[Asset Manager contract][asset-manager-contract] to apply changes to the validators' assets.

## Validator Registration

To become a validator, the first step is to register to the Validator Manager with at least `MIN_REGISTER_AMOUNT` of
KRO tokens. Upon registration, they initiate a [vault](asset-manager.md#composition-of-asset-manager) via
deposit and set commission rate and dedicated withdraw account. The commission rate is the
percentage that the validator takes from the output reward. When the validator requests for commission rate change,
he must wait for `COMMISSION_CHANGE_DELAY_SECONDS` to elapse from the moment of requesting commission rate change.
This safeguard can help delegators undelegate before the commission rate change is applied.

The `Validator` struct that represents the information of the validator is defined as follows:

```solidity
struct Validator {
    bool isInitiated;
    uint8 noSubmissionCount;
    uint8 commissionRate;
    uint8 pendingCommissionRate;
    uint128 commissionChangeInitiatedAt;
}
```

After registration, validators can receive delegations of KRO and KGH. When total deposited and delegated KRO exceeds
`MIN_ACTIVATE_AMOUNT`, validators can be activated and become eligible to submit output. Note that if the validators
don't meet the activation condition during registration and aren't automatically activated, the validator will need to
activate manually.

## Validator Management

Once initiated, validators are managed using a data structure called validator tree and categorized by statuses to
select priority validator and check output submission eligibility. Below are the implementation details for managing
validators.

### Bond

Validator System V2 brings the similar bond mechanism as the previous system. The validator must deposit a portion of
KRO deposited as a bond every time they submit an output or create a challenge. Bond amount for each output submission
or challenge creation is defined as `BOND_AMOUNT`. If there is not enough deposited KRO to bond, the validator will no
longer be able to submit outputs or create challenges.

### Validator Tree

To select next priority validator based on the amount of KRO delegated to the validator, validators are managed using a
[self-balancing binary search tree structure][self-balancing-binary-search-tree]. When a validator is activated, it is
inserted into the tree. The interface of the validator tree is shown below:

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

As described in [Priority Validator Selection](overview.md#priority-validator-selection), a validator's weight is
calculated by summing KRO deposited by validator, KRO delegated by KRO delegators, and cumulative reward.
It must be updated after each delegation, undelegation, slashing, and reward distribution. The next
priority validator is randomly selected from the validator tree based on their weight.

[self-balancing-binary-search-tree]: https://github.com/yasharpm/Solidity-Weighted-Random-List

### Status of Validator

Validator statuses are categorized as follows:

| Status       | initiated | activated | `MIN_REGISTER_AMOUNT` filled | `MIN_ACTIVATE_AMOUNT` filled |
|--------------|-----------|-----------|------------------------------|------------------------------|
| `NONE`       | X         | X         | X                            | X                            |
| `EXITED`     | O         | O/X       | X                            | O/X                          |
| `REGISTERED` | O         | X         | O                            | X                            |
| `READY`      | O         | X         | O                            | O                            |
| `INACTIVE`   | O         | O         | O                            | X                            |
| `ACTIVE`     | O         | O         | O                            | O                            |

Only validators with an `ACTIVE` status can submit outputs.

### Jail

If a validator has not submitted output when it was selected as a priority validator, the validator's
`noSubmissionCount` is incremented by 1. If the validator submits output the next time it is selected as a priority
validator, the `noSubmissionCount` is reset to 0. Otherwise, if the `noSubmissionCount` exceeds `JAIL_THRESHOLD`, the
validator is deemed to have failed to fulfill its obligations. Accordingly, the validator is sent to jail during
`SOFT_JAIL_PERIOD` and removed from the validator tree.

Additionally, the validator will be jailed if they submit incorrect outputs or challenge incorrectly. In other words, a
loser of challenge will result in being jailed for the `HARD_JAIL_PERIOD`. This `HARD_JAIL_PERIOD` is longer than the
`SOFT_JAIL_PERIOD` because such actions are considered threatening to the assets on Layer 2.

After each jail period has elapsed, the validator can be unjailed by calling `tryUnjail` function on its own and
activated if the activation conditions are met.

## Entry Point

Validator Manager contract acts as an entry point for external contracts to slash challenge losers or distribute output
rewards. Below is a detailed description of each case.

### Slashing

Slashing occurs during the challenge process. The assets subject to slashing are the bonds for output submission or
challenge creation. Namely, the delegated KROs from KRO delegators are not subject to slashing. When a challenge
concludes, the bond of the loser is slashed, with a portion imposed as tax to the security council and the remainder
awarded to the validator who won the challenge.

When the bond is slashed, [Colosseum](../../zk-fault-proof/challenge.md#contract-interface) contract calls Validator
Manager, which calls Asset Manager's slash function to deduct the bond from the validator's assets.

### Reward Distribution

Each time a submitted output is finalized, the reward for submitting the output is distributed to the output submitter.
As mentioned in [Reward Distribution](overview.md#reward-distribution), each reward is calculated as follows:

| Rewards For Validation                                             | Rewards For KRO Delegations         | Rewards For KGH Delegations            |
|--------------------------------------------------------------------|-------------------------------------|----------------------------------------|
| $\verb#BASE_REWARD# \times c_i + \verb#BOOSTED_REWARD# \times c_i$ | $\verb#BASE_REWARD# \times (1-c_i)$ | $\verb#BOOSTED_REWARD# \times (1-c_i)$ |

Here, $c_i$ is the commission rate of the validator. Note that a portion of the base reward and the boosted reward is
given as the validator reward through the commission. The remaining boosted rewards are shared among KGH delegators. The
remaining base rewards are shared among KRO delegators and the validator, proportional to the amount of KRO delegated
and deposited.

In addition, any pending challenge rewards accumulated as a result of slash are given to the validator who has won the
challenge.

## Summary of Definitions

### Constants

| Name                              | Value       | Unit    |
|-----------------------------------|-------------|---------|
| `MIN_REGISTER_AMOUNT`             | `10_000`    | KRO     |
| `MIN_ACTIVATE_AMOUNT`             | `10_000`    | KRO     |
| `COMMISSION_CHANGE_DELAY_SECONDS` | `1_209_600` | seconds |
| `SOFT_JAIL_PERIOD`                | `259_200`   | seconds |
| `HARD_JAIL_PERIOD`                | `604_800`   | seconds |
| `JAIL_THRESHOLD`                  | `2`         | number  |
| `BOND_AMOUNT`                     | `5_000`     | KRO     |
| `BASE_REWARD`                     | `400`       | KRO     |

# ETH-based Validator System

<!-- All glossary references in this file. -->

[g-validator]: ../../glossary.md#validator
[g-l2-output]: ../../glossary.md#l2-output-root

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Validator Pool Smart Contract](#validator-pool-smart-contract)
  - [Validation Rewards](#validation-rewards)
  - [Configuration of ValidatorPool](#configuration-of-validatorpool)
- [Summary of Definitions](#summary-of-definitions)
  - [Constants](#constants)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Validator Pool Smart Contract

Only accounts registered as [Validator][g-validator] can submit [output][g-l2-output] to
the [L2 Output Oracle](../validation#l2-output-oracle-smart-contract).
To register as a [Validator][g-validator], you must deposit at least `REQUIRED_BOND_AMOUNT` of ETH into
the `ValidatorPool` contract.
When submitting the output, the validator must bond Ethereum for `REQUIRED_BOND_AMOUNT`, which will be unbonded and
rewarded to the L2 `ValidatorRewardVault` contract when the output is finalized.

Also, validators should stake their bond for disputing challenge. This bond will be given to the winner of the challenge
as a reward. When this reward distributed, a [tax](../../fault-proof/challenge.md) is imposed to prevent collusive
attacks of asserter and challenger.

Validator Pool Smart Contract implements the following interface:

```solidity
interface ValidatorPool {
    /**
     * @notice Emitted when a validator bonds.
     *
     * @param submitter   Address of submitter.
     * @param outputIndex Index of the L2 checkpoint output index.
     * @param amount      Amount of bonded.
     * @param expiresAt   The expiration timestamp of bond.
     */
    event Bonded(
        address indexed submitter,
        uint256 indexed outputIndex,
        uint128 amount,
        uint128 expiresAt
    );

    /**
     * @notice Emitted when the pending bond is added.
     *
     * @param outputIndex Index of the L2 checkpoint output.
     * @param challenger  Address of the challenger.
     * @param amount      Amount of bond added.
     */
    event PendingBondAdded(uint256 indexed outputIndex, address indexed challenger, uint128 amount);

    /**
 * @notice Emitted when the bond is increased.
     *
     * @param outputIndex Index of the L2 checkpoint output.
     * @param challenger  Address of the challenger.
     * @param amount      Amount of bond increased.
     */
    event BondIncreased(uint256 indexed outputIndex, address indexed challenger, uint128 amount);

    /**
     * @notice Emitted when the pending bond is released(refunded).
     *
     * @param outputIndex  Index of the L2 checkpoint output.
     * @param challenger   Address of the challenger.
     * @param recipient    Address to receive amount from a pending bond.
     * @param amount       Amount of bond released.
     */
    event PendingBondReleased(
        uint256 indexed outputIndex,
        address indexed challenger,
        address indexed recipient,
        uint128 amount
    );

    /**
     * @notice Emitted when a validator unbonds.
     *
     * @param outputIndex Index of the L2 checkpoint output.
     * @param recipient   Address of the recipient.
     * @param amount      Amount of unbonded.
     */
    event Unbonded(uint256 indexed outputIndex, address indexed recipient, uint128 amount);

    /**
     * @notice Deposit ETH to be used as bond.
     */
    function deposit() external payable;

    /**
     * @notice Withdraw a given amount.
     *
     * @param _amount Amount to withdraw.
     */
    function withdraw(uint256 _amount) external;

    /**
     * @notice Bond asset corresponding to the given output index.
     *         This function is called when submitting output.
     *
     * @param _outputIndex Index of the L2 checkpoint output.
     * @param _expiresAt   The expiration timestamp of bond.
     */
    function createBond(
        uint256 _outputIndex,
        uint128 _expiresAt
    ) external;

    /**
     * @notice Adds a pending bond to the challenge corresponding to the given output index and challenger address.
     *         The pending bond is added to the bond when the challenge is proven or challenger is timed out,
     *         or refunded when the challenge is canceled.
     *
     * @param _outputIndex Index of the L2 checkpoint output.
     * @param _challenger  Address of the challenger.
     */
    function addPendingBond(uint256 _outputIndex, address _challenger) external;

    /**
     * @notice Releases the corresponding pending bond to the given output index and challenger address
     *         if a challenge is canceled.
     *
     * @param _outputIndex  Index of the L2 checkpoint output.
     * @param _challenger   Address of the challenger.
     * @param _recipient    Address to receive amount from a pending bond.
     */
    function releasePendingBond(
        uint256 _outputIndex,
        address _challenger,
        address _recipient
    ) external;

    /**
     * @notice Increases the bond amount corresponding to the given output index by the pending bond amount.
     *         This is when taxes are charged, and note that taxes are a means of preventing collusive attacks by
     *         the asserter and challenger.
     *
     * @param _outputIndex Index of the L2 checkpoint output.
     * @param _challenger  Address of the challenger.
     */
    function increaseBond(uint256 _outputIndex, address _challenger) external;

    /**
     * @notice Attempt to unbond. Reverts if unbond is not possible.
     */
    function unbond() external;

    /**
     * @notice Returns the balance of given address.
     *
     * @param _addr Address of validator.
     *
     * @return Balance of given address.
     */
    function balanceOf(address _addr) external view returns (uint256);

    /**
     * @notice Determines who can submit the L2 output next.
     *
     * @return The address of the validator.
     */
    function nextValidator() public view returns (address);
}
```

### Validation Rewards

A validator who submits an output can receive a reward from L2 `ValidatorRewardVault` contract when the output is
finalized, it is called validation reward. When the output is finalized, the `ValidatorPool` contract sends a message
to pay the reward in the L2 `ValidatorRewardVault` via the `KromaPortal` contract.
The reward is calculated by `ValidatorRewardVault` divided by `REWARD_DIVIDER`, which is the number of outputs
in a week.
Rewards received by the validator can be withdrawn to L1 via `withdraw()` in the ValidatorRewardVault.

### Configuration of ValidatorPool

`ROUND_DURATION` is equal to `(L2_BLOCK_TIME * SUBMISSION_INTERVAL) / 2`.

## Summary of Definitions

### Constants

| Name                   | Value | Unit           |
|------------------------|-------|----------------|
| `REQUIRED_BOND_AMOUNT` | `0.2` | ETH            |
| `REWARD_DIVIDER`       | `168` | num of outputs |

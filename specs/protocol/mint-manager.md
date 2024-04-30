# MintManager

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Activation](#activation)
- [Calculating Minting Amount](#calculating-minting-amount)
- [Initial Minting](#initial-minting)
- [Distribution](#distribution)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<!-- All glossary references in this file. -->

[g-l1-attr-deposit]: ../glossary.md#l1-attributes-deposited-transaction
[g-l1-attr-predeploy]: ../glossary.md#l1-attributes-predeployed-contract

## Overview

The `GovernanceToken` of Kroma is minted every block by the `MintManager` contract on L2 and distributed to recipients.
`MintManager` is a predeployed contract on L2 at address `0x4200000000000000000000000000000000000070`.

In the [L1 Attributes Deposited Transaction][g-l1-attr-deposit], the
[L1 Attributes Predeployed Contract][g-l1-attr-predeploy] calls the mint function of the MintManager contract to mint
tokens.

> Note that the mint function of the MintManager contract can only called by the
[L1 Attributes Predeployed Contract][g-l1-attr-predeploy]

## Activation

The mint function is activated from a specific L2 block number,
which is recorded in an immutable variable named `MINT_ACTIVATED_BLOCK`.
If the mint function is called before the activation block number, nothing will happen (it will not revert).

## Calculating Minting Amount

The amount of `GovernanceToken` minted per block decreases periodically.
The amount to be minted is calculated by repeatedly applying decay based on the current block number for a
certain number of epochs.
An epoch can be calculated by dividing the current block number by `SLIDING_WINDOW_BLOCKS` value.

Only up to 8 decimal places are considered for the minting amount.

```python
def get_mint_amount_per_block(block_number):
    epoch = (block_number - 1) // SLIDING_WINDOW + 1
    offset = (block_number - 1) % SLIDING_WINDOW + 1

    mint_amount = INIT_MINT_PER_BLOCK
    for i in range(1, epoch):
        mint_amount = (mint_amount * DECAYING_FACTOR) // DECAYING_DENOMINATOR
        mint_amount = mint_amount // FLOOR_UNIT * FLOOR_UNIT

    return mint_amount
```

- `SLIDING_WINDOW_BLOCKS`: The period (in blocks) which the mint amount decreases.
- `INIT_MINT_PER_BLOCK`: The amount minted per block during the first epoch.
- `DECAYING_FACTOR`: The ratio by which the mint amount decreases from the previous period.
- `DECAYING_DENOMINATOR`: The denominator of the decaying factor. This value is `10^5`.
- `FLOOR_UNIT`: The unit for rounding down decimal places. Since the decimal places for `GovernanceToken` are 18,
  and the mint amount is considered up to 8 decimal places, this value is `10^10`

## Initial Minting

Mints the amount of tokens that should have been minted from the L2 genesis block to the current block.
This function operates only if it has never been minted through the MintManager contract before.

```python
def initial_mint_amount(block_number):
    amount = 0
    mint_per_block = INIT_MINT_PER_BLOCK
    epoch = (block_number - 1) // SLIDING_WINDOW + 1
    offset = (block_number - 1) % SLIDING_WINDOW + 1

    for i in range(1, epoch):
        amount = amount + mint_per_block * SLIDING_WINDOW
        mint_per_block = (mint_per_block * DECAYING_FACTOR) // DECAYING_DENOMINATOR
        mint_per_block = (mint_per_block // FLOOR_UNIT) * FLOOR_UNIT

    if offset > 0:
        amount = amount + mint_per_block * offset

    return amount
```

## Distribution

After deploying the MintManager contract, the recipient addresses and their respective share percentages
are set during initialization.
The minted amount for the current block is then distributed to the configured addresses.

```python
mint_amount_per_block = get_mint_amount_per_block(block_number)

for i in range(len(recipients)):
    recipient = recipients[i]
    shares = shares[i]
    amount = mint_amount_per_block * shares / SHARE_DENOMINATOR
    
    governance_token_mint(recipient, amount)

last_minted_block = block_number
```

- `SHARE_DENOMINATOR`: The denominator of the share. This value is `10^5`.

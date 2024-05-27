# Validator System V2

<!-- All glossary references in this file. -->
[g-l2-output]: ../glossary.md#l2-output-root
[g-validator]: ../glossary.md#validator
[g-zk-fault-proof]: ../glossary.md#zk-fault-proof
[g-validator-pool-contract]: ../glossary.md#validator-pool-contract

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Background](#background)
- [Overview](#overview)
- [Contents](#contents)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Background

The current permissionless [validator][g-validator] system of Kroma Network ensures economic security by
bonding 0.2 ETH each time a [L2 output root][g-l2-output] is submitted. While successfully involving over 360
validators (as of 6/3/24), there are a few drawbacks in the current system that need to be addressed:

- **Economic Security**: There are some rooms to improve the economic security of current system, as submitting a
  malicious output result in a loss of up to 0.2 ETH.
- **Network Participation**: Despite having one of the largest validator pools among current L2 networks, only those
  capable of running validators can participate, excluding many who wish to contribute to network security and earn
  incentives due to technical barriers.

These are addressed by introducing economic security and network participation improvements in the new validator
system. Similar to Delegated Proof of Stake (DPoS) systems, the new validator system will fortify the security of the
entire network and lower the barrier of participation, using the token delegation system based on Kroma Network's
governance token (KRO) and Kroma Guardian House (KGH) NFT.

## Overview

**Validator System V2** is a new network security model of Kroma, which incorporates KRO tokenomics to improve the
current system. It retains current [ZK fault proof][g-zk-fault-proof] and challenge system, while introducing two
new contracts: **Validator Manager** and **Asset Manager** contract. These contracts replace the existing
[Validator Pool contract][g-validator-pool-contract], handling jobs such as validator registration, output submitter
selection and probability calculation, delegation, slashing, and reward distribution.

In Validator System V2, KRO token holders can stake tokens to become validators or delegate tokens to active
validators, contributing to network security. KGH NFT holders can stake their NFTs to boost validator selection
probability and share rewards.

## Contents

- Specifications
  - [Asset Manager](./asset-manager.md)

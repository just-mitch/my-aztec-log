# Introduction

Presently, if a transaction is reverted, the base rollup does not mark it as reverted: it merely has its revertible side effects dropped. This issue will update the base rollup to mark reverted transactions.

# Solution Design

## New `RevertCode` in `TxEffect`

Create a new class in `circuits.js` to hold a `RevertCode`. It will be a safe wrapper around an `Fr` that will be used to store the revert code of a transaction.

`0` will indicate success, and any other value will indicate failure. Presently, only `1` is used to indicate general failure, but we'll leave the door open for future expansion.

`TxEffect` will be updated to include a `RevertCode`.

### Ethereum `status`

Ethereum stores their `status` in a `uint64`, but only currently use `0` to indicate failure and `1` for success. [More info](https://github.com/ethereum/EIPs/issues/98) (Thanks @spalladino !)

### Size considerations

We'll eventually want to add `gasUsed` and `gasPrice` to `TxEffect`. Using a full field is wasteful, but using a smaller size will make the encoding and hashing more complex (see below). We'll implement with a full field and have a stacked PR to optimize the size.

## Kernel Constraints

The `RevertCode` will become part of the content commitment for a transaction.

The content commitment is computed by the base rollup in `compute_tx_effects_hash`, which:

- accepts a `CombinedAccumulatedData`
- operates over `Field`s

We will therefore update the `CombinedAccumulatedData` to include the `RevertCode`.

A fallout from that is we will move the current `reverted` flag from `PublicKernelCircuitPublicInputs` to be part of:

- `PrivateAccumulatedNonRevertibleData`
- `PublicAccumulatedNonRevertibleData`

This is because we:

1. build up a `CombinedAccumulatedData` during private execution
2. split them into `PrivateAccumulatedNonRevertibleData` and `PrivateAccumulatedRevertibleData` in the private kernel tail
3. convert those `PrivateAccumulated...` into `PublicAccumulated...` during public kernel execution as needed
4. recombine them back into `CombinedAccumulatedData` before feeding back into base rollup

## Encoding

The `RevertCode` will come first in the encoded `TxEffect`.

That is we will publish:

```
|| 32 bytes for RevertCode || ... Existing PublishedTxEffect ... ||
```

## L1 Tx Decoder

We'll need to update the availability oracle to compensate for the new flag.

## `TxReceipt`

We'll add a new `TxStatus` that is `REVERTED = 'reverted'`.

We need to update `getSettledTxReceipt` to inspect the `TxEffect` and set the `status` accordingly.

# Test Plan

## Unit Tests

- Serde of `TxEffect` in TS and solidity.

## E2E Tests

- Add checks for tx statuses of `REVERTED` in the `e2e_fees`, `e2e_deploy_contract`, and `dapp_testing`.

# Documentation Plan

Will start a page in the "Data publication and availability" section of the yellow paper describing the format of the block submitted.

# Especially Relevant Parties

- @PhilWindle
- @benesjan
- @alexghr
- @LeilaWang
- @LHerskind

# Plan Approvals

**_👋 Please add a +1 or comment to express your approval or disapproval of the plan above._**

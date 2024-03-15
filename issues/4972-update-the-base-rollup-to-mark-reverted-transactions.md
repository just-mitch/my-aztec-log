# Introduction

Presently, if a transaction is reverted, the base rollup does not mark it as reverted: it merely only has its revertible side effects dropped. This issue will update the base rollup to mark reverted transactions.

# Solution Design

## Augment `TxEffect`

Add to `TxEffect`:

- `reverted`: a `bool`

### Ethereum `status`

Ethereum stores their `status` in a `uint64`, but only currently use `0` to indicate failure and `1` for success. We don't need to repeat the same mistake. [More info](https://github.com/ethereum/EIPs/issues/98) (Thanks @spalladino !)

### Future additions

We'll eventually want to add `gasUsed` and `gasPrice` to `TxEffect`. We'll add them to `RollupKernelCircuitPublicInputs` when we do.

## Kernel Constraints

The `reverted` flag will become part of the content commitment for a transaction.

We can just add the `reverted` flag to `RollupKernelCircuitPublicInputs`.

### No public component

Note that the `reverted` flag comes from the `PublicKernelCircuitPublicInputs`. If we're coming straight from private, we'll manually set the flag to false now, since in the future we'll probably need a specialized circuit to handle the case where there is no public component. @LeilaWang ?

## Encoding

The `reverted` flag will come first in the encoded `TxEffect`.

That is we will publish:

```
|| 1 bit for reverted || ... Existing PublishedTxEffect ... ||
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

Will start a page in the "Data publication and availability" section of the yellow paper describing the format of the block submitted and the `reverted` flag.

# Especially Relevant Parties

- @PhilWindle
- @benesjan
- @alexghr
- @LeilaWang
- @LHerskind

# Plan Approvals

**_👋 Please add a +1 or comment to express your approval or disapproval of the plan above._**

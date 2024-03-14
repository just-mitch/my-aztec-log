# Introduction

Presently, if a transaction is reverted, the base rollup does not mark it as reverted: it merely only has its revertible side effects dropped. This issue will update the base rollup to mark reverted transactions.

# Solution Design

## `PublishedTx`

The data that presently gets included on chain is captured in `TxEffect`. We want to add _metadata_ in this case. Further, we want to future proof in that we will eventually want to include information such as `gasUsed` and/or `gasPrice` in the future.

Considering:

- a `Tx` is the proven output from _just_ local/private execution.
- a `TxReceipt` is the L2 interface for dropped/pending/mined transactions
- a `TxEffect` is the data that actually gets published to DA for a TX

I propose to rename `TxEffect` to `PublishedTxEffect`, and add two new classes:

- `PublishedTxMetadata` which will contain the published metadata for a transaction
- `PublishedTx` which will contain the `PublishedTxEffect` and `PublishedTxMetadata`

For now, `PublishedTxMetadata` will only contain:

- `reverted`: a `bool`

Note: eth stores their `status` in a `uint64`, but only currently use `0` to indicate failure and `1` for success. No idea why they chose such a large number.

## Kernel Constraints

The `PublishedTxMetadata` will need to be part of the content commitment for a transaction.

For now, we can just add the `reverted` flag to `RollupKernelCircuitPublicInputs`. When we add `gasUsed` and `gasPrice` in the future, we will need to add them to `RollupKernelCircuitPublicInputs` as well, so at that point we might add a noir equivalent to `PublishedTxMetadata` and include that in `RollupKernelCircuitPublicInputs` instead.

Note that the reverted flag comes from the `PublicKernelCircuitPublicInputs`. If we're coming straight from private, we'll manually set the flag to false now, but in the future we'll probably need a specialized circuit to handle the case where there is no public component.

## Encoding

Adding these TS abstractions should not bloat the on-chain encoding of the data. The `PublishedTx` will be encoded as a `TxEffect` is today, with the `PublishedTxMetadata` encoded as a single byte.

That is we will publish:

```
|| 1 bit for reverted || ... Existing PublishedTxEffect ... ||
```

## L1 Tx Decoder

We'll need to update the availability oracle to compensate for the `PublishedTxMetadata`.

## `TxReceipt`

We'll add a new `TxStatus` that is `REVERTED = 'reverted'`.

We need to update `getSettledTxReceipt` to inspect the `PublishedTxMetadata` and set the `status` accordingly.

# Test Plan

## Unit Tests

- Serde of `PublishedTxMetadata` and `PublishedTx` in TS and solidity.

## E2E Tests

- Add checks for tx statuses of `REVERTED` in the `e2e_fees`, `e2e_deploy_contract`, and `dapp_testing`.

# Especially Relevant Parties

- @PhilWindle
- @benesjan
- @alexghr
- @LeilaWang

# Plan Approvals

**_👋 Please add a +1 or comment to express your approval or disapproval of the plan above._**

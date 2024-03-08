# Introduction

Presently, logs are emitted without a side effect counter, and they all go to the "revertible" side effect set, i.e.
`PrivateAccumulatedRevertibleData` if we are in private, and `PublicAccumulatedRevertibleData` if we are in public.

This poses a problem for transactions that revert during app logic, because we might have emitted logs in setup (e.g. because we unshielded) that we need to retain.

## Kernel Outputs

The `*AccumulatedRevertibleData` structs hold:

```ts
/**
 * Accumulated unencrypted logs hash from all the previous kernel iterations.
 * Note: Represented as a tuple of 2 fields in order to fit in all of the 256 bits of sha256 hash.
 */
public unencryptedLogsHash: Tuple<Fr, typeof NUM_FIELDS_PER_SHA256>,
/**
 * Total accumulated length of the encrypted log preimages emitted in all the previous kernel iterations
 */
public encryptedLogPreimagesLength: Fr,
```

Note the fact that 2 `Fr` are required [may be dropped soon](https://github.com/AztecProtocol/aztec-packages/issues/2019).

# Solution Design

Currently thinking it might be easier to just split effects into `end` and `endNonRevertible` as they are processed in the private kernel. Then we don't need to "add side effects" to the logs.

# Test Plan

Outline the unit tests that will be written for this feature, including the logic they cover and any mock objects used.
Describe the e2e tests that will be added.

# Documentation Plan

Identify changes or additions to the user documentation, or yellow-paper.

# Especially Relevant Parties

List contributors who may be interested in the issue and can review the plan.

# Plan Approvals

Please add a +1 or comment to express your approval or disapproval of the plan above.

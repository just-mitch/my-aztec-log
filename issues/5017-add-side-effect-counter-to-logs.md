# Introduction

Presently, logs are emitted without a side effect counter, and they all go to the "revertible" side effect set, i.e.
`PrivateAccumulatedRevertibleData` if we are in private, and `PublicAccumulatedRevertibleData` if we are in public.

This poses a problem for transactions that revert during app logic, because we might have emitted logs in setup (e.g. because we unshielded) that we need to retain.

Further, there is an issue in that without side effect counters, logs have an emitted ordering that is not "natural". E.g. in private execution

```rust
fn A() {
  // emit log0
  B();
  // emit log2;
}

fn B() {
  // emit log1;
}
```

The logs will be emitted as `[log0, log2, log1]`. This is because `B` is treated as a nested call of `A`, so when logs are pushed into its `ClientExecutionContext`, they are accumulated in the private kernel **after** the kernel iteration for `A` is complete, and thus after `A`'s logs.

Note, we carry forward information about each function executions logs within the kernel iteration as part of `AccumulatedRevertibleData`, as:

```rust
encrypted_logs_hash: [Field; NUM_FIELDS_PER_SHA256], // should be one Field [soon](https://github.com/AztecProtocol/aztec-packages/issues/2019)
unencrypted_logs_hash: [Field; NUM_FIELDS_PER_SHA256],

// Here so that the gas cost of this request can be measured by circuits, without actually needing to feed in the
// variable-length data.
encrypted_log_preimages_length: Field,
unencrypted_log_preimages_length: Field,
```

```rust
let previous_encrypted_logs_hash = public_inputs.end.encrypted_logs_hash;
let current_encrypted_logs_hash = private_call_public_inputs.encrypted_logs_hash;
// compute_logs_hash just concatenates the two hashes and hashes them
public_inputs.end.encrypted_logs_hash = compute_logs_hash(previous_encrypted_logs_hash,current_encrypted_logs_hash);
```

The first step here is to just get side effect counters associated with logs so that they wind up in the right order. Next step (see https://github.com/AztecProtocol/aztec-packages/issues/4712) will be splitting logs into revertible and non-revertible.

# Solution Design

We also don't want to mess with the construction/encoding of the log entries themselves because that data ultimately goes on chain and we don't want to leak privacy and increase the DA cost.

We also don't want to drop the flexibility of an arbitrary number of logs.

## WIP

Can we do something where in the oracle call to execute a private function, when we return, strip out the logs and add them to our context?

```ts
const childExecutionResult = await executePrivateFunction(
  context,
  targetArtifact,
  targetContractAddress,
  targetFunctionData
);

if (isStaticCall) {
  this.#checkValidStaticCall(childExecutionResult);
}

// Absorb the new logs
this.encryptedLogs.push(...childExecutionResult.encryptedLogs.logs);
childExecutionResult.encryptedLogs = FunctionL2Logs.empty();

//
this.nestedExecutions.push(childExecutionResult);
```

# Test Plan

Outline the unit tests that will be written for this feature, including the logic they cover and any mock objects used.
Describe the e2e tests that will be added.

# Documentation Plan

Identify changes or additions to the user documentation, or yellow-paper.

# Especially Relevant Parties

List contributors who may be interested in the issue and can review the plan.

# Plan Approvals

Please add a +1 or comment to express your approval or disapproval of the plan above.

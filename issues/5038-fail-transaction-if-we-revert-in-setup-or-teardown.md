# Introduction

The work done in https://github.com/AztecProtocol/aztec-packages/issues/4971 made it so that if a transaction reverts while executing the "app logic" phase of a transaction, we no longer drop the transaction and instead include it but drop all the revertible side effects.

This work ensures that we do in fact **drop** the transaction if it gives an error in any non-revertible phases, as this would render the transaction invalid.

For more context, see https://github.com/AztecProtocol/aztec-packages/issues/4096.

# Solution Design

## Updates to the public processor and phase managers

At present, we `publicStateDB.commit()` at the end of each phase. This is a problem because if we revert in the teardown phase, we will have already committed the setup phase.

But we need to somehow commit the setup phase, because if we revert in the app logic phase, we need to keep the setup phase.

Broadly, we need to be able to "soft commit" to changes that occur during a phase.

I will call that "soft commit" a "checkpoint".

We then need the ability to `rollback` either to the last `checkpoint`, or to the last `commit`.

To do this, we can modify the `PublicStateDB` interface:

```ts
/**
 * Mark the uncommitted changes in this TX as a checkpoint.
 */
checkpoint(): Promise<void>

/**
 * Rollback to the last checkpoint.
 */
rollbackToCheckpoint(): Promise<void>

/**
 * Commit the changes in this TX. Includes all changes since the last commit,
 * even if they haven't been covered by a checkpoint.
 */
commit(): Promise<void>

/**
 * Rollback to the last commit.
 */
rollbackToCommit(): Promise<void>
```

Under the hood in `WorldStatePublicDB`, all updates will still initially go into the `uncommitedWriteCache`, but we'll just add a new `checkpointedWriteCache`. Only downside is that `storageRead` will need to check both caches, but they should stay small, so it should be fine.

In the `setup_phase_manager` and `teardown_phase_manager` we can do:

```ts
const [
  publicKernelOutput,
  publicKernelProof,
  newUnencryptedFunctionLogs,
  revertReason,
] = await this.processEnqueuedPublicCalls(
  tx,
  previousPublicKernelOutput,
  previousPublicKernelProof
).catch(
  // the abstract phase manager throws if simulation gives error in a non-revertible phase
  async (err) => {
    await this.publicStateDB.rollbackToCommit();
    throw err;
  }
);
tx.unencryptedLogs.addFunctionLogs(newUnencryptedFunctionLogs);
await this.publicStateDB.checkpoint();
return { publicKernelOutput, publicKernelProof, revertReason };
```

In `app_logic_phase_manager`, we can do:

```ts
const [
  publicKernelOutput,
  publicKernelProof,
  newUnencryptedFunctionLogs,
  revertReason,
] = await this.processEnqueuedPublicCalls(
  tx,
  previousPublicKernelOutput,
  previousPublicKernelProof
).catch(
  // if we throw for any reason other than simulation, we need to rollback and drop the TX
  async (err) => {
    await this.publicStateDB.rollbackToCommit();
    throw err;
  }
);

if (revertReason) {
  await this.publicContractsDB.removeNewContracts(tx);
  await this.publicStateDB.rollbackToCheckpoint();
} else {
  tx.unencryptedLogs.addFunctionLogs(newUnencryptedFunctionLogs);
  await this.publicStateDB.checkpoint();
}
return { publicKernelOutput, publicKernelProof, revertReason };
```

And finally in `tail_phase_manager`:

```ts
const [publicKernelOutput, publicKernelProof] = await this.runKernelCircuit(
  previousPublicKernelOutput,
  previousPublicKernelProof
).catch(
  // the abstract phase manager throws if simulation gives error in non-revertible phase
  async (err) => {
    await this.publicStateDB.rollbackToCommit();
    throw err;
  }
);

// commit the state updates from this transaction
await this.publicStateDB.commit();

return { publicKernelOutput, publicKernelProof, revertReason: undefined };
```

## Circuit changes

As a sanity check, we need to verify in the public kernel setup and teardown that the public call did not revert.

# Test Plan

## Unit Tests

Add to `public_processor.test.ts`: show that if a transaction reverts in the setup or teardown phase, it is "Failed" and not "Reverted".

Add to `world_state_public_db.test.ts`: show that the `rollbackToCheckpoint` and `rollbackToCommit` methods work as expected.

Add to setup and teardown public kernel circuits to show that they fail assertions if the public call is `reverted`.

## E2E Tests

In `e2e_fees.test.ts`, we can add bugged `FeePaymentMethod`s that:

- take too much of a fee in setup
- distribute too much of a fee in teardown

# Especially Relevant Parties

- @alexghr
- @PhilWindle
- @LeilaWang

# Plan Approvals

Please add a +1 or comment to express your approval or disapproval of the plan above.

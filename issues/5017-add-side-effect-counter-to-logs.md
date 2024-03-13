# Introduction

Presently, logs are not published in their chronological order.

For example,

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

The logs are presently be emitted as on chain `[log0, log2, log1]`.

The goal of this work is to instead emit the logs in their "chronological" order, `[log0, log1, log2]`.

# Non-goals

This issue is just focused on emitting logs in their "chronological" order and producing valid blocks.
There is a follow on issue for [splitting into revertible and non-revertible logs](https://github.com/AztecProtocol/aztec-packages/issues/4712)

# Solution criteria

Requirements:

- make all (un)encrypted logs available in the L2 block body appear in the same chronological order as they were emitted
- don't leak any additional privacy
- don't lose the current flexibility of an arbitrary number of logs
- don't make additional trust assumptions
- easily extensible to public execution
- easily extensible to [splitting into revertible and non-revertible logs](https://github.com/AztecProtocol/aztec-packages/issues/4712)

Nice to haves:

- don't increase DA cost
- don't blow up circuit sizes

# Additional background

- [Information on the original encoding scheme.](https://discourse.aztec.network/t/proposal-forcing-the-sequencer-to-actually-submit-data-to-l1/426)
- [Known trust issue producing the log hash](https://github.com/AztecProtocol/aztec-packages/issues/1165)

# Current state

I'll focus on encrypted logs, and show how they are created and flow through the system.

## Start private simulation - top level

When starting simulation at the top level (see `simulator.ts#run`), we create a fresh `ClientExecutionContext`, which has

```ts
//... other stuff
private encryptedLogs: Buffer[] = [];
private nestedExecutions: ExecutionResult[] = [];
```

This context is passed to `private_execution.ts#executePrivateFunction` to fulfill oracle calls. For our purposes, we care about calls to `emit*ncryptedLog` and `callPrivateFunction`

`executePrivateFunction` returns an `ExecutionResult`, which has:

```ts
export interface ExecutionResult {
  // ... other stuff
  callStackItem: PrivateCallStackItem;
  nestedExecutions: this[];
  encryptedLogs: FunctionL2Logs;
}

export class FunctionL2Logs {
  constructor(
    /**
     * An array of logs.
     */
    public readonly logs: Buffer[],
  ) {}
  // ...
}

class PrivateCallStackItem {
  // ... other stuff
  public publicInputs: PrivateCircuitPublicInputs,
}


class PrivateCircuitPublicInputs {
  // ... other stuff
  public encryptedLogsHash: [Field, Field]; // NOTE these aren't actually get set by the private circuit. See below.
  public encryptedLogPreimagesLength: Field;
}
```

## Emitting logs

When a function emits an encrypted log (from aztec-nr), it ultimately hits the oracle call in `client_execution_context.ts#emitEncryptedLog`:

```ts
public emitEncryptedLog(contractAddress: AztecAddress, storageSlot: Fr, noteTypeId: Fr, publicKey: Point, log: Fr[]) {
  const note = new Note(log);
  const l1NotePayload = new L1NotePayload(note, contractAddress, storageSlot, noteTypeId);
  const taggedNote = new TaggedNote(l1NotePayload);
  // There is also a TODO in
  const encryptedNote = taggedNote.toEncryptedBuffer(publicKey, this.curve);
  this.encryptedLogs.push(encryptedNote);
}
```

Note that nothing is returned from this function.

## Nested calls

When a private function calls a private function, we

```ts
const childExecutionResult = await executePrivateFunction(
  // note the recursive call
  context, // a new context
  targetArtifact,
  targetContractAddress,
  targetFunctionData
);
// ...
this.nestedExecutions.push(childExecutionResult);
```

## `executePrivateFunction`

Here, we get our acir bytecode and initial witness from the circuit artifact and context (which provides the args) and pass them to the acvm, which gives us a partial witness, which we can use in combination with the artifact to get `PrivateCircuitPublicInputs`.

**BUT**, we don't actually use the `PrivateCircuitPublicInputs` to get our log hash or length. Instead, we have:

```ts
const publicInputs = PrivateCircuitPublicInputs.fromFields(returnWitness);

const encryptedLogs = context.getEncryptedLogs(); // just returns its own encrypted logs, now as a `FunctionL2Logs`
const unencryptedLogs = context.getUnencryptedLogs(); // same
// TODO(https://github.com/AztecProtocol/aztec-packages/issues/1165) --> set this in Noir // <- this TODO is in the source

// presently, this hash is over e.g. using the example above,  || log0_length || log0 || log2_length || log2 ||
publicInputs.encryptedLogsHash = to2Fields(encryptedLogs.hash());

publicInputs.encryptedLogPreimagesLength = new Fr(
  encryptedLogs.getSerializedLength()
);
publicInputs.unencryptedLogsHash = to2Fields(unencryptedLogs.hash());
publicInputs.unencryptedLogPreimagesLength = new Fr(
  unencryptedLogs.getSerializedLength()
);
```

## Private simulation proving

After private simulation is complete (see `pxe_service.ts#simulateAndProve`), we have our single `ExecutionResult`, which has its own `encryptedLogs` and `nestedExecutions`.

For example, we could have an `ExecutionResult` that looks like:

```ts
{
  // ...
  encryptedLogs: FunctionL2Logs {
    logs: Buffer[] // e.g. [log0, log2]
  },
  nestedExecutions: [
    {
      // ...
      encryptedLogs: FunctionL2Logs {
        logs: Buffer[] // e.g. [log1]
      },
      nestedExecutions: [
        // ...
      ]
    }
  ]
}
```

In `kernel_prover.ts`, we create an execution stack formed initially from the single `ExecutionResult`.

While the stack is not empty, we pop an execution off the top (which becomes the `currentExecution`), and push its nested executions.

We turn that `ExecutionResult` and some other stuff into a `PrivateCallData`, and the only thing that has on it that we care about are the `PrivateCircuitPublicInputs`, which have the `encryptedLogsHash` and `encryptedLogPreimagesLength` that we set in `executePrivateFunction`.

Then, since this is the first `ExecutionResult` we are processing, we pass this `PrivateCallData` (and `TxRequest`, which has nothing interesting for us at the moment) into the private kernel init.

## Private kernel init

The private kernel init (and private kernel inner) want to build

```rust
struct PrivateKernelInnerCircuitPublicInputs {
    // other stuff
    end: CombinedAccumulatedData,
}

struct CombinedAccumulatedData {
    // other stuff
    encrypted_logs_hash: [Field; NUM_FIELDS_PER_SHA256], // should be one Field [soon](https://github.com/AztecProtocol/aztec-packages/issues/2019)
    encrypted_log_preimages_length: Field,
}
```

So it creates an empty builder `let mut public_inputs: PrivateKernelCircuitPublicInputsBuilder = unsafe::zeroed();`

and we copy the information from our private call into the kernel outputs as

```rust
let previous_encrypted_logs_hash = public_inputs.end.encrypted_logs_hash; // zero for the private kernel init
let current_encrypted_logs_hash = private_call_public_inputs.encrypted_logs_hash; // the hash we set in executePrivateFunction
// compute_logs_hash just concatenates the two hashes and hashes them
// so in our case it would be hash( zero || hash ( log0_length || log0 || log2_length || log2 ) )
public_inputs.end.encrypted_logs_hash = compute_logs_hash(previous_encrypted_logs_hash,current_encrypted_logs_hash);

public_inputs.end.encrypted_log_preimages_length = public_inputs.end.encrypted_log_preimages_length +
                                                        private_call_public_inputs.encrypted_log_preimages_length;
```

## Private kernel inner

Very similar. As input we now have:

```rust
struct PrivateKernelInnerCircuitPrivateInputs {
    previous_kernel: PrivateKernelInnerData,
    private_call: PrivateCallData, // constructed from the next ExecutionResult
}

struct PrivateKernelInnerData {
    // other stuff
    public_inputs : PrivateKernelInnerCircuitPublicInputs,
}
```

This time, however, we initialize our output builder with our previous kernel's output:

```rust
let mut public_inputs : PrivateKernelCircuitPublicInputsBuilder = unsafe::zeroed();
// ...
let start = previous_kernel.public_inputs.end;

public_inputs.end.encrypted_logs_hash = start.encrypted_logs_hash;
public_inputs.end.encrypted_log_preimages_length = start.encrypted_log_preimages_length;
```

So when we update our `encrypted_logs_hash`, we would have

```rust
let previous_encrypted_logs_hash = public_inputs.end.encrypted_logs_hash; // the hash we set in the private kernel init
let current_encrypted_logs_hash = private_call_public_inputs.encrypted_logs_hash; // the hash we set in executePrivateFunction

// so in our case it would be hash( hash( zero || hash ( log0_length || log0 || log2_length || log2 ) ) || hash ( log1_length || log1 ) )
public_inputs.end.encrypted_logs_hash = compute_logs_hash(previous_encrypted_logs_hash,current_encrypted_logs_hash);

public_inputs.end.encrypted_log_preimages_length = public_inputs.end.encrypted_log_preimages_length +
                                                        private_call_public_inputs.encrypted_log_preimages_length;
```

## Private kernel tail

Basically a no-op for logs. The `encrypted_logs_hash` and `encrypted_log_preimages_length` are passed through to

```rust
struct PrivateKernelTailCircuitPublicInputs {
    end: PrivateAccumulatedRevertibleData, // here
}
```

## Private log accumulation

After proving, back in `pxe_service.ts#simulateAndProve`, we

```ts
// reach recursively into the nested executions and collect all the logs
// NOTE: this is using the top level execution result
const encryptedLogs = new TxL2Logs(collectEncryptedLogs(executionResult));

// where collectEncryptedLogs is
// [execResult.encryptedLogs, ...[...execResult.nestedExecutions].reverse().flatMap(collectEncryptedLogs)]

// so we have a TxL2Logs that looks like
// {
//   functionLogs: [
//     {
//       logs: [log0, log2]
//     },
//     {
//       logs: [log1]
//     }
//   ]
// }

return new Tx(
  publicInputs,
  proof,
  encryptedLogs,
  unencryptedLogs,
  enqueuedPublicFunctions
);
```

## Solo block builder

## Base rollup circuit

## L2 block encoding

A `TxL2Logs` is presently encoded/serialized into the L2 block body as follows:

```
tx_logs_len || f1_logs_len || f1_log1_len || f1_log1 || f1_log2_len || f1_log2 || ... || f2_logs_len || ...
```

where `tx_logs_len` is the total length of all logs emitted in the transaction, `f1_logs_len` is the total length of all logs emitted in the first function, and so on.

## L1 verification

# Proposed solution

We cannot encode the logs in the L2 block body grouped by function, because as we've seen, they could overlap. So we need to encode them in the order they were emitted.

We will present the L2 block encoding, and work backwards down the stack to arrive at how such an encoding can be produced.

## L2 block encoding

We will simplify the encoding to just:

```
tx_logs_len || log1_len || log1 || log2_len || log2 || ...
```

where `log1_len` is the length of `log1`, and `log1` is the first chronologically emitted log, and so on.

# Test Plan

Outline the unit tests that will be written for this feature, including the logic they cover and any mock objects used.
Describe the e2e tests that will be added.

# Documentation Plan

Identify changes or additions to the user documentation, or yellow-paper.

# Especially Relevant Parties

List contributors who may be interested in the issue and can review the plan.

# Plan Approvals

Please add a +1 or comment to express your approval or disapproval of the plan above.

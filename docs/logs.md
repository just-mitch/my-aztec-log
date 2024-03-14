# State of logs 2024-03-14

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

## ProcessedTx

When finished public execution, we convert the `Tx` into a `ProcessedTx`:

```ts
interface ProcessedTx {
  encryptedLogs: TxL2Logs;
  data: PublicKernelCircuitPublicInputs;
}

export class PublicKernelCircuitPublicInputs {
  private combined: CombinedAccumulatedData | undefined = undefined;
  constructor(
    public endNonRevertibleData: PublicAccumulatedNonRevertibleData,
    public end: PublicAccumulatedRevertibleData
  ) {}

  get combinedData() {
    if (!this.combined) {
      this.combined = CombinedAccumulatedData.recombine(
        this.endNonRevertibleData,
        this.end,
        this.reverted
      );
    }
    return this.combined;
  }
}

//...
function recombine(
  nonRevertible: PublicAccumulatedNonRevertibleData,
  revertible: PublicAccumulatedRevertibleData,
  reverted: boolean
): CombinedAccumulatedData {
  return new CombinedAccumulatedData(
    // presently, all logs are revertible
    revertible.encryptedLogsHash,
    revertible.encryptedLogPreimagesLength
  );
}
```

## Solo block builder - runCircuits

`SoloBlockBuilder.buildL2Block` takes in an array of `ProcessedTx`.

For each, we `buildBaseRollupInput(tx: ProcessedTx, globalVariables: GlobalVariables)`, which inserts leaves into trees, and converts our `ProcessedTx` into a

```rs
struct RollupKernelCircuitPublicInputs {
    // other stuff
    end: CombinedAccumulatedData,
}
```

## Base rollup circuit

The `base_rollup_inputs.nr` produces a `BaseOrMergeRollupPublicInputs`, and the only thing on it we care about is `txs_effects_hash : [Field; NUM_FIELDS_PER_SHA256],`.

This hash is computed in `noir-projects/noir-protocol-circuits/crates/rollup-lib/src/components.nr#compute_txs_effects_hash`. It concatenates all the note commitments, nullifiers, etc, _and_ the `encrypted_logs_hash` from the `ProcessedTx` and `sha256`'s the result.

## Merge rollup circuit

The `merge_rollup_inputs.nr` also produces a `BaseOrMergeRollupPublicInptus`, and computes its `tx_effects_hash` by concatenating the `tx_effects_hash` from two base rollups and `sha256`'ing the result.

## Root rollup

The `root_rollup_inputs.nr` produces a `RootRollupPublicInputs`:

```rust
struct RootRollupPublicInputs {
    // other stuff
    header: Header,
}

struct Header {
    // other stuff
    content_commitment: ContentCommitment
}

struct ContentCommitment {
    // other stuff
    txs_effects_hash: [Field; NUM_FIELDS_PER_SHA256], // concatenates the tx_effects_hash from the two previous merge rollups and hashes the result
}
```

## Solo block builder - buildL2Block

After the call from `runCircuits`, we have our `RootRollupPublicInputs` and array of `ProcessedTx`. We convert each `ProcessedTx` into a `TxEffect`:

```ts
export function toTxEffect(tx: ProcessedTx): TxEffect {
  return new TxEffect(
    // ...
    tx.encryptedLogs || new TxL2Logs([]),
    tx.unencryptedLogs || new TxL2Logs([])
  );
}
```

Which we use to produce a `Body`:

```ts
export class Body {
  constructor(
    public l1ToL2Messages: Tuple<
      Fr,
      typeof NUMBER_OF_L1_L2_MESSAGES_PER_ROLLUP
    >,
    public txEffects: TxEffect[]
  ) {}
}
```

The `Body` and the `RootRollupPublicInputs` is then used to produce a `L2Block`:

```ts
const l2Block = L2Block.fromFields({
  archive: circuitsOutput.archive,
  header: circuitsOutput.header, // holds the content commitment we computed in the root rollup
  body: blockBody, // holds the raw logs
});
```

We then sanity check the hashes:

```ts
if (
  !l2Block.body
    .getTxsEffectsHash()
    .equals(circuitsOutput.header.contentCommitment.txsEffectsHash)
) {
  throw new Error(...);
}
```

The `getTxsEffectsHash` mirrors the merkle tree construction in the circuits. Its leaves are comprised of `this.txEffects.map(txEffect => txEffect.hash());`.

In the course of computing `txEffect.hash()`, we call `txEffect.encryptedLogs.hash()`, which mirrors the construction of the `encrypted_logs_hash` in the private kernel, i.e.

```ts
const logsHashes: [Buffer, Buffer] = [Buffer.alloc(32), Buffer.alloc(32)];
let kernelPublicInputsLogsHash = Buffer.alloc(32);

for (const logsFromSingleFunctionCall of this.functionLogs) {
  logsHashes[0] = kernelPublicInputsLogsHash;
  logsHashes[1] = logsFromSingleFunctionCall.hash(); // privateCircuitPublicInputsLogsHash

  // Hash logs hash from the public inputs of previous kernel iteration and logs hash from private circuit public inputs
  kernelPublicInputsLogsHash = sha256(Buffer.concat(logsHashes));
}

return kernelPublicInputsLogsHash;
```

And `logsFromSingleFunctionCall.hash()` is

```ts
// Remove first 4 bytes that are occupied by length which is not part of the preimage in contracts and L2Blocks
const preimage = this.toBuffer().subarray(4);
return sha256(preimage);
```

## L1 body encoding and publishing

in `L1Publisher#processL2Block`, we

```ts
const encodedBody = block.body.toBuffer();
// ...
const txHash = await this.sendPublishTx(encodedBody);
```

Which ultimately writes the encoded body to our AvailabilityOracle contract on L1.

A `TxL2Logs` is presently encoded/serialized into the L2 block body as follows:

```
tx_logs_len || f1_logs_len || f1_log1_len || f1_log1 || f1_log2_len || f1_log2 || ... || f2_logs_len || ...
```

where `tx_logs_len` is the total length of all logs emitted in the transaction, `f1_logs_len` is the total length of all logs emitted in the first function, and so on.

The availability oracle calls `bytes32 txsEffectsHash = TxsDecoder.decode(_body);`.

This walks through the same operations as the `txEffect.hash()` during:

```solidity
function publish(bytes calldata _body) external override(IAvailabilityOracle) returns (bytes32) {
  bytes32 txsEffectsHash = TxsDecoder.decode(_body);
  isAvailable[txsEffectsHash] = true;

  emit TxsPublished(txsEffectsHash);

  return txsEffectsHash;
}
```

## L1 processing

Once the body is published, we actually process it, by calling

```ts
const processTxArgs = {
  header: block.header.toBuffer(),
  archive: block.archive.root.toBuffer(),
  body: encodedBody,
  proof: Buffer.alloc(0),
};

// ...
const txHash = await this.sendProcessTx(processTxArgs);
```

This calls out to our Rollup contract and calls `process`.

This extracts the content commitment from the header, `header.contentCommitment.txsEffectsHash = bytes32(_header[0x0044:0x0064]);` which, if you recall, is the `txsEffectsHash` we computed in the root rollup.

It then ensures that the `txsEffectsHash` that was committed to is what was published earlier:

```solidity
if (!AVAILABILITY_ORACLE.isAvailable(header.contentCommitment.txsEffectsHash)) {
  revert Errors.Rollup__UnavailableTxs(header.contentCommitment.txsEffectsHash);
}
```

A few more checks later and it calls `emit L2BlockProcessed(header.globalVariables.blockNumber);`.

Et voila.

This issue serves as the parent issue (epic) for all issues pertaining to the work needed to get fees operational for the purposes of the "End to End Internal Testnet" (ITN).

# Functional Requirements

## Enshrined token and bridge

There will be an enshrined gas paying token on L2, and an enshrined 1-way bridge from L1 to L2.
The enshrined gas token will have no functionality beyond being a token that can be used to pay for gas on L2.

## 3-phase transactions and non-revertible side effects

Transactions will be conceptually broken into three phases:

- setup
- app logic
- teardown

When performing private execution, users will have the ability to create "non-revertible" side effects.
If the user enqueues public calls that are non-revertible, the public call that is enqueued last is treated as the call that will be executed for "teardown" by the sequencer. The remaining non-revertible enqueued public calls are run as part of setup by the sequencer. All revertible public calls are run as part of app logic by the sequencer.

## Fee Payment Contracts

We will write a canonical FPC (fee payment contract) that will allow users to pay fees without using the enshrined gas token directly. These contracts specify a list of tokens they support, a commission, and can handle public or private payments, as well as public or private refunds.

## Reverts

If the app logic phase reverts, all revertible side effects (including those from private execution) will be dropped, the setup and teardown phases will still be executed, and their (non-revertible) side effects as well as the non-revertible side effects from private execution will be retained.

Metadata concerning the transaction status will be included in the block.

## Sequencer checks

The sequencer will have the ability to inspect functions that are enqueued for setup/teardown to decide if it wants to take the risk to execute the transaction: if these functions do in fact revert, the transaction is invalid and cannot be added to the chain, thus the sequencers will not be paid for the work they did.

## Metering and rewards

Transactions will be able to specify:

- a DA gas limit
- a fee per DA gas
- a L2 gas limit
- a fee per L2 gas

The base rollup will make assertions that the total supply of the enshrined gas token is reduced by `DA_gas_used * fee_per_DA_gas + L2_gas_used * fee_per_L2_gas` for each transaction that is executed. The merge rollup will aggregate these sums, and the root rollup will ensure they are included as part of the block rewards.

Metadata concerning, limits, consumed gas, and fees will be included in the block.

```[tasklist]
### Tasks
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/5001
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/4998
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/4096
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/4999
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/5002
```

This issue serves as the parent issue (epic) for all issues pertaining to the work needed to get harden the node (and the protocol circuits it runs) to an acceptable point for the purposes of the "End to End Internal Testnet" (ITN).

# Circuit Trust

Presently there are instances where the public kernel circuit and rollup circuits place undue trust in the sequencer, such as squashing public data writes, and recombining revertible/non-revertible side effects.

# Client Exploits

We need to probe for basic exploits that the node/sequencer might be able to perform on the client, including:

- overcharging fees
- manipulating transactions

# P2P Network

We need to determine an acceptable level of security for our P2P network, such as how/if we will protect against various attacks (e.g. eclipse, sybil, etc.)

```[tasklist]
### Tasks
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/5011
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/5014
- [ ] https://github.com/AztecProtocol/aztec-packages/issues/5015
```

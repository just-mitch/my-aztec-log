# On Tap

# 2024-04-05

## Focus

- [x] make updates to gas accounting writeup
- [x] schedule review of gas accounting writeup
- [ ] read mikes thread on note discovery
- [ ] start spike homomorphic implementation of partial notes

# 2024-04-04

## Focus

- [x] share writeup on gas accounting

## Extra

- [ ] Finish interfaces chapter in rust for rustaceans

# 2024-04-03

## Focus

- [ ] fix merge conflicts in PR
  - started, stopped after big merge from Leila
- [x] read through slack
- [x] start writeup on structs that need to change in the kernel w.r.t. gas accounting

## Extra

- [x] continue survey paper on [commitment schemes and zk protocols](https://homepages.cwi.nl/~schaffne/courses/crypto/2014/papers/ComZK08.pdf?utm_source=pocket_saves)

# 2024-03-28

## Focus

- [x] post PR of private DA metering

## Extra

- [x] finish types chapter in rust for rustaceans
- [x] review existing interface for aztec merkle tree:s

# 2024-03-27

## Focus

- [x] continue implementation of private DA metering

## Extra

- [x] read zk modular stack paper from lisa
- [x] read 4844 spec again

# 2024-03-26

## Focus

- [x] Clear review stack
- [x] continue implementation of DA metering

## Extra

- [ ] finish types chapter in rust for rustaceans
- [ ] define interface for rust merkle trees

# 2024-03-25

## Focus

- [ ] Attempt to clean up intellisense.
  - Failure
- [x] Meet with a16z research on fees
  - great conversation, especially with Pranav afterward. key takeaways
  - nailing down l1 verification cost is the focus.
  - can use shapley to get to 1 variable for l1 verification costs, but need a way to either
    - constrain sequencer to commit to a "cost" ahead of accepting bids, or
    - have a proxy pay out the refunds
  - if we convert to blobs for DA, we can amortize that as well
  - until we have serious contention for l2 blockspace, we can just have a fixed multiple for coming up with an l2 price, based on metering.
- [x] continue implementation of DA metering

## Extra

- [ ] started survey paper on [commitment schemes and zk protocols](https://homepages.cwi.nl/~schaffne/courses/crypto/2014/papers/ComZK08.pdf?utm_source=pocket_saves)

# 2024-03-24

## Accomplishments

- Implemented revert code
- Design for DA gas profiler
- Thorough reviews of proving single private function inclusion and dynamic proving
- Volunteered for security work

# 2024-03-22

## Charlie Donut

- [x] Security position/work?
  - Will pass on to Kesha
- [x] Concerns about quantum?
  - Expects evolutions in our crypto. Should have enough lead time

## Focus

- [x] post design for gas profiler

# 2024-03-21

## Reviews

- [x] Phil's dynamic prover
- [~] A bunch from Palla
  - did the base PR.

## Focus

- [ ] post design for gas profiler
  - reviews took a long time to digest.
  - got a rough diagram sketched.
- [ ] read material on parity circuits

# 2024-03-20

## Focus

- skim strategy docs
  - [x] noir commercial goals
  - [x] growth goals
  - [x] developer funnel
  - [x] execution environment engineering
- [x] merge 'mark-reverted-transactions'
- [ ] post design for DA gas profiler - made great exploratory progress

# 2024-03-19

## Focus

- [x] update design comment on 'mark-reverted-transactions'
- [ ] merge 'mark-reverted-transactions' - couldn't because of bug on master
- [x] add design for more efficient encoding of revertCode
- [x] start design for da gas profiler

## Reviews

- Alex
  - [x] [whitelisting FPC work](https://github.com/AztecProtocol/aztec-packages/pull/5310)
  - [x] [move entrypoints back into accounts.js so we can reuse their code](https://github.com/AztecProtocol/aztec-packages/pull/5312)
  - [x] [add signerless entrypoint contract, update signerless wallet, deploy gas token with it](https://github.com/AztecProtocol/aztec-packages/pull/5313)

# 2024-03-18

## Focus

- [x] get 4972 passing existing tests
- [x] get 4972 passing new tests

## Reviews

- review phil's prover work
  - [x] https://github.com/AztecProtocol/aztec-packages/pull/5259
- review alex's work on sequencer checks
  - [x] https://github.com/AztecProtocol/aztec-packages/pull/5265
  - [x] https://github.com/AztecProtocol/aztec-packages/pull/5266
  - [x] https://github.com/AztecProtocol/aztec-packages/pull/5267

## Phil 1-1

- I volunteered to focus on security, Phil is going to get that ball rolling
- Likely going to use calldata for DA for ITN
- Likely need native merkle tree ops- concern with using rust because our hashes are written in c++ so either the FFI needs to be good or we need to rewrite the hashes in rust

# 2024-03-15

- [x] share https://hackmd.io/8Nj4_P3aSMiHn871JOIuOQ
- [x] review https://github.com/AztecProtocol/aztec-packages/pull/5236
- [ ] implement https://github.com/AztecProtocol/aztec-packages/pull/5226 on '4972-update-the-base-rollup-to-mark-reverted-transactions'

# 2024-03-14

- [x] digest yp docs on logs and ordering in circuits
- [x] meet with Jan, Leila, and Miranda about attacking a larger log refactor
- [x] update log issues in github
- [x] review https://github.com/AztecProtocol/aztec-packages/issues/5209
- [x] post design for https://github.com/AztecProtocol/aztec-packages/pull/5226

# 2024-03-13

TODO:

- [x] Re-review Alex's https://github.com/AztecProtocol/aztec-packages/pull/5129
- [x] Re-review Alex's https://github.com/AztecProtocol/aztec-packages/pull/4891
- [x] Send initial thoughts on ordered logs to Jan and Leila

DROPPED:

- Post plan for https://github.com/AztecProtocol/aztec-packages/pull/5017
  Realizing this is way to big to be a single PR plan. Need a more holistic approach.

STANDUP:

Discussed possible issue with not having up to date trees while simulating in public.
Voiced opinion that integrating the proving system is more important than native merkle trees.

# 2024-03-12

TODO:

- [x] Talk with Nico about how to plumb max_block_number through the circuits
- [x] Review issue from Alex https://github.com/AztecProtocol/aztec-packages/issues/5150
- [x] Review Alex's https://github.com/AztecProtocol/aztec-packages/pull/5129
- [ ] Post plan for https://github.com/AztecProtocol/aztec-packages/pull/5017

DROPPED:

- Review Palla's https://github.com/AztecProtocol/aztec-packages/pull/5141

# 2024-03-11

TODO:

- [x] Implement https://github.com/AztecProtocol/aztec-packages/pull/5038
- [x] Continue plan for https://github.com/AztecProtocol/aztec-packages/pull/5017

# 2024-03-08

TODO:

- [x] Review Palla's https://github.com/AztecProtocol/aztec-packages/pull/5059
- [x] Review Alex's post in aztec3-fees
- [x] Review Alex's https://github.com/AztecProtocol/aztec-packages/pull/5030
- [x] Review mem leak https://github.com/AztecProtocol/aztec-packages/pull/5023
- [x] Voiced concerned on https://github.com/AztecProtocol/aztec-packages/pull/4891
- [x] Post plan for https://github.com/AztecProtocol/aztec-packages/pull/5038
- [x] read https://aztec.network/blog/regeneration-a-manifesto-for-an-autonomous-future/
- [x] read https://forum.aztec.network/t/how-to-do-contract-upgrades-feb-2024-edition/4583/3
- [x] read https://forum.aztec.network/t/proposal-forcing-the-sequencer-to-actually-submit-data-to-l1/426/1
- [x] read https://github.com/AztecProtocol/aztec-packages/pull/4891#issuecomment-1986272222
- [x] Start plan for https://github.com/AztecProtocol/aztec-packages/pull/5017

BLOCKED:

- [Awaiting plan approval] Implement https://github.com/AztecProtocol/aztec-packages/pull/5038

# 2024-03-07

TODO:

- [x] review https://github.com/AztecProtocol/aztec-packages/pull/4983
- [x] review https://github.com/AztecProtocol/aztec-packages/pull/5030
- [x] review https://github.com/AztecProtocol/aztec-packages/pull/4910
- [x] merge https://github.com/AztecProtocol/aztec-packages/pull/5034
- [x] review alex's post in aztec-3
- [x] create template for PR descriptions
- [x] started plan for #5038

# Standup notes

Phil was working on fixing the release. Going to create an epic for doing native merkle trees.
Alex is going to respond to questions about the genericity of his approach on partial notes, and post in aztec-3 asking for feedback on side-effect counters in public.
Spyros tracked down the memory leak to an issue with interruptible sleeps.
Palla is continuing to remove dead code, and adjust solidity contract code to reflect changes to the block.
Mitch got basic public reverts to PR. Found flakiness in a test that he raised to Palla who is going to submit a fix. Plans to start adding side effect counters to logs.

---

# 2024-03-06

# Standup Notes

## Phil

- fixing build system and deploy for sandbox
- acvm ready

## Spyros

- investigating mem leak in node. possibly in memory fifo

## Palla

- killing dead code
- spoke to leila about moving teams

## Alex

- figure out why dapp sub wasn't working
- tracked to problem with non/revertible side effects
- tried quick fix, but not easy since pub kernel checks side effects from private
- any pub calls that enqueue a pub call have side effects of zero

## Mitch

- code reviews, got all test passing for public reverts
- new test added for public reverts fails
- might be related to Alex's issue
- if it is, plans to post PR as is, new test disabled, with follow on issue to address the problem

## Action Items

- [x] Phil, Mitch, and Alex to catch up about fees

# Fee Catchup

- [ ] Mitch to add new issue encapsulating the fee work. abandon "phases", focus on functional requirements and relative priorities

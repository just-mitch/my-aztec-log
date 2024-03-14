# 2024-03-15

- [ ] share https://hackmd.io/8Nj4_P3aSMiHn871JOIuOQ
- [ ] review https://github.com/AztecProtocol/aztec-packages/pull/5201

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

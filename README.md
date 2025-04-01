# Multi-Paxos Implementation
> _Note: Due to course and DSLabs policy, no source code is included. The implementation details are described conceptually._

## Overview

As part of a graduate-level course on distributed systems, I implemented Multi-Paxos in Java, drawing inspiration from the **Paxos Made Moderately Complex (PMMC)** paper [[source](https://www.cs.cornell.edu/courses/cs7412/2011sp/paxos.pdf)]. The foundation of the project is the [DSLabs framework](https://github.com/emichael/dslabs), which is tailored for testing and simulating distributed protocols.

In contrast to the role-based architecture prescribed by PMMC—where nodes function separately as leaders, acceptors, and replicas—I adopted a unified server design. Each `PaxosServer` dynamically assumes all roles simultaneously. This architectural simplification significantly reduces coordination overhead and facilitates tighter integration of protocol phases.

Key features:
- Single-threaded, event-driven execution model
- Ballot-based leader election and leader stability guarantees
- Full implementation of Phase 1 and Phase 2 of Paxos
- Fine-tuned heartbeat intervals and election backoffs
- Decision synchronization and log garbage collection


## Design Highlights

### Unified Role Model

PMMC splits server functionality into three interacting roles: **leader**, **acceptor**, and **replica**. In my implementation, all nodes are identical and execute a single class, `PaxosServer`. Internally, each server tracks its role based on context: whether it is currently the leader, or participating in leader election. This role unification eliminates role-based message routing and results in a cleaner logic flow. 


### Lexicographic Ballot Numbers
Ballot numbers in my implementation consist of two components: an incrementing integer round and the unique leader Address. Ballots are compared lexicographically, first by the round number, then by leader address to deterministically break ties. This strategy guarantees a strict ordering of ballots, essential for leader stability.

### Phase 1 and 2 Synchronization

In Multi-Paxos, the protocol is divided into two distinct phases that ensure safe consensus. **Phase 1** (also known as the *Prepare phase*) occurs exclusively during leader elections. In this phase, the candidate leader sends `Phase1a` messages to collect previously accepted slot values from a **majority** of acceptors. Receiving responses (`Phase1b`) from a majority—defined as at least half of all acceptors plus one—ensures no conflicting decisions have been made previously. This quorum requirement is crucial because, in general, a system with \(2x + 1\) nodes can safely tolerate up to \(x\) failures without losing consistency or availability. 

Upon successfully completing Phase 1, the elected leader transitions into **Phase 2** (the *Accept phase*), where it proposes commands and drives them to consensus by again receiving acceptances (`Phase2b`) from a majority of acceptors. Any in-between unused slots identified after Phase 1 are explicitly filled with `NO_OP_COMMAND` and committed via Phase 2 to preserve log consistency. Thus, both phases work together to maintain strict safety guarantees, ensuring consensus correctness despite faults or partial failures in the distributed system.

### Leader Stability

Correctness in Paxos is based on majority agreement, but **liveness** depend on electing and maintaining a stable leader. Only one active leader can propose values for slots. If all nodes continuously attempt to become leader (a scenario called livelock), the system stalls. To address this, the implementation uses heartbeats and adaptive timers:
- The current leader sends periodic `Heartbeat` messages with its ballot and log position.
- Acceptors reset their `ElectionTimer` upon receipt.
- If no heartbeat is received before timeout, a acceptor increments its ballot and begins a new election.

The distinction between **heartbeat interval** and **election timeout** is critical. A short heartbeat interval ensures rapid confirmation, while a longer election timeout prevents premature challenges.


### Messages and Timers
- `Phase1a`: Initiate Phase 1 during election
- `Phase1b`: Accepted responses with past ballots
- `Phase2a`: Proposal for a slot
- `Phase2b`: Acceptance of proposal
- `Decision`: Confirm a value has been chosen for a slot
- `PaxosRequest`: Client request for command execution
- `PaxosReply`: Client response with result
- `Heartbeat` / `HeartbeatReply`: Leader liveness + sync
- `GarbageCollect`: Log cleanup notification
- `HeartbeatTimer`: Scheduled emission of heartbeats
- `ElectionTimer`: Timeout to initiate leader election
- `ClientTimer(currentCommand)` Retry client requests


### Log Organization with Slot Invariants

The Multi-Paxos protocol relies on a replicated log indexed by slot numbers. Each slot represents a consensus instance. We maintain two integer counters:
- `slot_out`: the next slot index to be executed in-order
- `slot_in`: the next available slot index for proposing a new command

This distinction is crucial. While `slot_in` tracks proposal progress, `slot_out` ensures sequential consistency. Slot entries can be in states such as `EMPTY`, `ACCEPTED`, `CHOSEN`, or `CLEARED` (i.e., garbage collected). A central invariant is that execution proceeds only when slot `s` is `CHOSEN` and all slots `< s` have been executed. 


Gaps (i.e., committed slots appearing after uncommitted ones) are common. Because chosen slots may arrive out of order, execution is performed in a loop that checks whether the next `slot_out` has been committed:

```text
while slotMap[slot_out].state == CHOSEN:
    execute(slot_out)
    slot_out++
```


### Decision Syncing via Heartbeat Replies

To keep replicas consistent, each `Heartbeat` includes the leader’s current `slot_out`. Each acceptor replies with its own `slot_out`, allowing the leader to determine if the acceptor is lagging and send `Decision` messages.

```text
on HeartbeatReply from acceptor:
    if acceptor.slot_out < leader.slot_out:
        send Decision(acceptor.slot_out)
```

This proactive synchronization ensures all nodes eventually converge.

### Garbage Collection: Paxos’s Log Compaction

This same mechanism also enables safe log compaction. Once the leader has collected `slot_out` from all acceptors, it computes:

```text
min_slot = min(acceptor.slot_out for all acceptors)
```

All slots `< min_slot` have been executed and can be discarded. The leader broadcasts `GarbageCollect(min_slot)` to propagate this to all nodes.

This garbage collection mechanism is akin to Raft’s snapshotting: it bounds memory usage without affecting correctness. It also aligns with the theoretical requirement that Paxos must eventually forget decisions once they are durable.


---

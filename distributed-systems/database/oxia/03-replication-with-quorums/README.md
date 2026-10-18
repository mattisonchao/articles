<!--
HANDOFF NOTES (remove before publishing)
- Series: part 3 of "Designing a Distributed Metadata System" (LinkedIn).
  Part 1 "Foundation" (2026-07-05), part 2 "From Local State to a Replicated State Machine" (2026-07-26).
- Status: draft approved by the author on 2026-10-16. Remaining work is fact-checking (REVIEW markers below) and optional trimming.
- Agreed order: unreliable channel -> replication -> consensus -> Paxos / Raft. Byzantine tolerance is explicitly out of scope (see "Which failures do we tolerate?").
- Style to keep, matching parts 1 and 2: ## and ### headings, a bolded one-line takeaway opening each section, prose and bullets, no code, tables or diagrams, no product names (cite papers instead).
- Length: about 4,500 words, longer than parts 1 and 2. If it needs cutting, move "Multi-Paxos" and "Flexible quorums" to part 4 (election).
- Part 4 (next): leader election and fencing. Its outline is in "Towards leader election".
-->

# Designing a Distributed Metadata System: Replication with Quorums

In the previous article, we turned a local metadata store into a replicated state machine. A leader assigned log positions, followers persisted the entries, and an entry counted as committed once a majority had it. We said this works "assuming a valid leader exists" and moved on.

That sentence hid most of the difficulty. This article goes back and builds the theory behind it, in three steps:

- **The channel.** What a network between replicas actually guarantees, and why that makes distribution hard.
- **Replication.** How to keep copies of the log durable over such a channel, and why quorums are the answer.
- **Consensus.** Why replication turns into an agreement problem, and how Paxos and Raft solve it.

Leader election is the subject of the next article. Here, we still assume a single leader per term and focus on what makes replication safe.

## Why the network changes everything

**On one machine, a function call either returns or the process dies. Across machines, a message can vanish, and you cannot tell why.**

A standalone metadata server has a simple failure model. If the process is alive, a write either succeeds or returns an error. If the process crashes, it recovers from its WAL and snapshot. Every component shares one clock, one memory, and one fate.

Distribution breaks all three. Replicas run on different machines, observe time differently, and fail independently. The only thing connecting them is the message channel, and that channel is far weaker than a function call. To reason about replication, we first need a model of what the channel and the processes can do.

### A system model

**Protocols are only correct relative to a model, so the model has to be stated first.**

Michel Raynal's *Fault-Tolerant Message-Passing Distributed Systems* builds its algorithms on a small, explicit model, and it is a good frame for metadata systems:

- **Processes** are a fixed set of n nodes. Up to t of them may crash. A crashed process stops and takes no further steps.
- **Asynchrony** means there is no bound on how long a process takes to execute a step, and no bound on how long a message takes to arrive. <!-- REVIEW: check the CAMP_n,t[∅] notation and its wording against Raynal's book. -->

Raynal names this crash-prone asynchronous message-passing model CAMP_n,t[∅], where the empty brackets mean "no additional assumptions."
- **Channels** connect pairs of processes. What a channel guarantees is the interesting part.

This model is pessimistic on purpose. Real networks are usually fast, but "usually" is not a correctness argument. A protocol that is safe in the asynchronous model stays safe during a GC pause, a saturated NIC, or a cross-zone partition, because none of those situations is worse than what the model already allows.

One consequence follows immediately. Without a bound on message delay, a process waiting for a reply cannot tell whether the peer is slow or dead. A timeout is a guess, and every guess is sometimes wrong. We will come back to this when we reach consensus.

### Reliable and unreliable channels

**A reliable channel is something we build, not something we get.**

The classic channel properties are stated for a message m sent from process p to process q:

- **No creation:** if q receives m, then p actually sent m.
- **No duplication:** q receives m at most once.
- **No loss:** if p sends m and q does not crash, q eventually receives m.

A channel with all three properties is *reliable*. Physical networks do not provide it. Packets are dropped by congested switches, duplicated by retransmissions, and reordered by different paths. TCP hides some of this within one connection, but a connection can break, and a client that reconnects has no idea which of its in-flight requests the other side processed.

Raynal works with weaker channels instead.

<!-- REVIEW: check that Raynal's fair-lossy definition matches this list, especially "finite duplication". --> The most useful one is the *fair-lossy* channel:

- It may lose messages.
- It does not create messages, and it duplicates a message only a finite number of times.
- **Fairness:** if p sends m infinitely often, q receives m infinitely often.

Fairness is the minimum promise that makes communication possible at all. A channel that could lose every message forever would be indistinguishable from a crashed peer.

On top of a fair-lossy channel, we can build a reliable one with two familiar mechanisms:

- **Retransmission** handles loss. The sender keeps resending m until it receives an acknowledgment.
- **Sequence numbers** handle duplication. The receiver remembers what it has delivered and discards repeats.

This looks easy, and it is the source of a surprising amount of complexity.

### Why the simple fix is not simple

**Retransmission needs to know when to stop, and in an asynchronous system that knowledge is not available.**

Retransmission alone raises three questions:

<!-- REVIEW: check the Aguilera-Chen-Toueg claim (quiescent reliable communication is impossible without failure detectors) and its citation venue and year (WDAG 1997?). -->

- **When can the sender stop?** Only after an acknowledgment arrives. If the receiver has crashed, no acknowledgment ever arrives, and the sender retransmits forever. Raynal calls a protocol *quiescent* if it eventually stops sending messages. Aguilera, Chen, and Toueg showed that quiescent reliable communication over lossy links is impossible in an asynchronous system with crashes, unless the processes have some failure-detection information. The sender needs some hint that the peer is dead to stop talking to it.
- **How much does the receiver remember?** Deduplication needs state. If sequence numbers are bounded and the channel can reorder messages, an old message can arrive looking like a new one. Real systems handle this with large, monotonic identifiers and by tying them to a session or epoch.
- **What did the other side do?** An acknowledgment is itself a message, and it can be lost too.

The last point leads to one of the oldest results in the field.

### The Two Generals problem

**Over a lossy channel, two processes can never be certain they agree.**

Two generals must attack at the same time, and they communicate only through messengers who may be captured. General A sends "attack at dawn." A cannot attack until it knows B received the message, so B sends an acknowledgment. Now B cannot be sure its acknowledgment arrived, so A must acknowledge the acknowledgment, and so on. Any finite protocol has a last message. If that message is lost, the sender of it cannot tell, and the receiver's decision cannot depend on it. By induction, no finite exchange produces certainty.

For a metadata store, the Two Generals problem appears every time a client writes:

- The client sends a write. The response times out.
- The write may have been lost before reaching the leader. It may have been applied and committed, with only the response lost. Or it may still be sitting in a queue.
- The client cannot distinguish these cases, and it must still decide whether to retry.

If the client retries a non-idempotent mutation, it may apply twice. If it gives up, it may report a failure for a write that succeeded. This is why the compare-and-set semantics from the first article matter so much. A version check executed inside the ordered apply path turns an ambiguous retry into a safe one: the duplicate fails its version check instead of silently overwriting state.

### What this means for our metadata store

**Every assumption the standalone server made silently now needs an explicit mechanism.**

Mapping the theory back to the system from the first two articles:

- **Lost requests and responses** require client retries, so mutations must be idempotent or guarded by versions.
- **Duplicated and reordered replication messages** require followers to identify each entry precisely and to accept it only if it fits the log they already have.
- **Unbounded delays** mean a leader cannot wait for every follower. It must make progress with a subset.
- **Undetectable crashes** mean no single replica's silence can block the system, and no single replica's reply can be trusted as the whole truth.

The last two points shape how we replicate.

## Replication over an unreliable channel

**Replication is about keeping enough copies that no failure we tolerate can erase a write.**

The goal of replication is durability and availability beyond one machine. A write acknowledged to a client must survive the crash of any replica, and the system must keep accepting writes while some replicas are down. Over the channel we just described, the obvious designs each fail one of these goals.

### Two designs that do not work

**Waiting for everyone sacrifices availability. Waiting for no one sacrifices durability.**

- **Write to all replicas before acknowledging.** Every acknowledged write is on every replica, so durability is excellent. But one crashed or slow replica stalls every write, and since a crash looks like slowness, the leader cannot even tell how long to wait. Availability is as weak as the weakest replica.
- **Write locally and replicate in the background.** Writes are fast and never block. But if the leader crashes before a follower receives the entry, an acknowledged write is lost. Durability is as weak as a single machine, which is exactly what we set out to fix.

Real systems sometimes choose one of these on purpose. Asynchronous replication is common for disaster-recovery copies, and some primary-backup systems wait for every replica in a membership list that an external manager shrinks when a replica fails. But for the core of a metadata store, we want something that tolerates failures without that kind of outside help.

### Quorums

**A majority can always vouch for any decision another majority made, because any two majorities share a member.**

The middle ground is to wait for a *quorum*: a subset of replicas large enough that its knowledge cannot be lost, and small enough that a minority of failures cannot block it.

The property that matters is **quorum intersection**. With n replicas, any two sets of more than n/2 replicas must overlap in at least one replica. This gives us a durability argument:

- An entry is committed once a majority has persisted it.
- Anyone who later needs to know what was committed asks a majority.
- Those two majorities share at least one replica, and that replica knows about the committed entry.

So a committed entry can always be found, no matter which replicas crash, as long as no more than a minority fails at the same time. To tolerate f failures, we need n = 2f + 1 replicas. Three replicas tolerate one failure, five tolerate two.

Quorums also fit the channel model well. A leader never waits for a specific replica, only for any majority. Lost messages are retransmitted, slow replicas are simply not among the first to answer, and crashed replicas are outvoted.

Majorities also explain why adding replicas does not improve write throughput. Each additional replica increases the quorum size, so a write waits for more acknowledgments. Replication buys fault tolerance, not scale. Scale comes from sharding, which is a later topic.

### Identifying entries

**A follower must be able to tell whether a message belongs in its log, using only the message itself.**

Because replication messages can be duplicated, delayed, and reordered, a follower cannot rely on arrival order. Each entry needs an identity that makes stale and repeated messages recognizable. A log position alone is not enough: two different leaders, at different times, might each write an entry at position 7. The entry therefore also carries the *term* (or *epoch*) of the leader that created it, a number that grows each time leadership changes.

With (term, position) on every entry, a follower can apply simple local rules:

- A message from an older term than the one it has already seen is stale and is rejected.
- A duplicated entry with the same (term, position) is applied idempotently.
- An entry that does not extend the follower's log is rejected, and the leader backs up and resends.

We will see this structure again in Raft.

### The hard part is not copying, it is deciding

**Quorums make a decision durable. They do not make sure there is only one decision.**

Quorum replication works cleanly while a single leader decides what goes into each log position. The trouble starts when the leader fails, or merely looks like it failed:

- The old leader may have replicated an entry to some replicas but not a majority. Is it committed or not?
- A new leader may write a different entry at the same position. Which one wins?
- A paused old leader may wake up and continue sending entries for the positions it believes it owns.

Each of these is a disagreement about what the log contains. Resolving it is no longer a durability problem. It is an agreement problem, and agreement among unreliable processes over unreliable channels has a name.

## What consensus is

**A replicated log is a sequence of agreement decisions, one per log position.**

Formally, consensus asks a set of processes to decide on one value among those proposed, with three properties:

- **Validity:** the decided value was proposed by some process.
- **Agreement:** no two processes decide different values.
- **Termination:** every correct process eventually decides.

A replicated log runs consensus once per log position. Position 1 decides the first mutation, position 2 the second, and so on. If every replica decides the same mutation at every position, deterministic apply from the previous article guarantees they all reach the same state. So the question "how do we replicate the log safely across leader changes?" is really "how do we solve consensus repeatedly and efficiently?"

### The FLP result

**In an asynchronous system, consensus cannot be guaranteed to both stay correct and always finish.**

Fischer, Lynch, and Paterson proved in 1985 that in an asynchronous system where even one process may crash, no deterministic algorithm can guarantee agreement *and* termination. This holds even with reliable channels. Lossy channels only make it worse.

The intuition comes from the slow-or-dead ambiguity. To finish, a protocol must eventually stop waiting for some process. But that process may only be slow, and its view may be the one that decides the outcome. Any rule for when to stop waiting can be defeated by a schedule of delays that keeps the system undecided.

FLP does not make consensus useless. It means a practical protocol must choose what it gives up when the network misbehaves. Paxos and Raft make the same choice:

- **Safety is unconditional.** Replicas never disagree on committed entries, however bad the network gets.
- **Liveness is conditional.** Progress is guaranteed only when the system behaves synchronously for long enough to keep a stable leader.

This split is the key to reading every protocol in this article. The rules exist to protect safety under the full asynchronous model. Timeouts, heartbeats, and leaders exist to recover liveness when the network is cooperative.

### Which failures do we tolerate?

**A metadata store inside one trust domain needs crash tolerance, not Byzantine tolerance, but it must be honest about what "crash" means.**

Consensus protocols differ mainly in the failures they assume. There are three common models:

- **Crash-stop.** A process stops and never returns. This is the model of FLP and of Raynal's CAMP_n,t[∅].
- **Crash-recovery.** A process stops and may later restart with whatever it persisted to stable storage. This is what real replicas do, and it is the model Paxos and Raft actually target.
- **Byzantine.** A faulty process may do anything: send conflicting messages to different peers, forge state, or lie about what it accepted.

Byzantine fault tolerance is expensive. Protocols such as PBFT need 3f + 1 replicas to tolerate f faulty ones instead of 2f + 1, extra message rounds, and cryptographic authentication of every message. They make sense when replicas are run by parties who do not trust each other, as in blockchains or multi-organization systems.

A metadata store is different. Its replicas run the same code, are deployed by the same operator, and live inside one security boundary. A replica that is fully compromised already has access to the data it would be lying about. For this setting, Paxos, Raft, ZooKeeper, etcd, and most production metadata systems tolerate crashes, not Byzantine behavior.

That does not mean we can ignore everything that is not a clean crash. The crash-recovery model relies on one promise: **a replica that restarts remembers everything it acknowledged.** Some failures quietly break that promise and turn an honest replica into one that behaves, from the protocol's point of view, like a liar:

- **Lost writes.** If a replica acknowledges an entry before it is truly durable, for example by skipping fsync, a power loss erases an entry that a quorum was counting on.
- **Disk replacement.** A replica that rejoins with an empty disk but the same identity may vote or acknowledge as if it never knew anything. If enough replicas do this, two majorities may no longer share a replica that remembers.
- **Silent corruption.** A flipped bit in a log segment or snapshot can change an entry without changing its position.

The defenses are cheaper than Byzantine tolerance, and the first article already introduced some of them:

- Persist entries and votes before acknowledging them.
- Checksum log records and snapshots, and refuse to serve corrupted data.
- Give each replica's storage an identity, and treat a replica with lost state as a new member that must catch up before it can count toward a quorum.

In short, we design for crash-recovery, defend against storage faults with checksums and identities, and leave Byzantine tolerance to systems whose replicas do not trust each other.

## Paxos: the core of safe agreement

**Paxos shows the minimum a protocol must do to decide one value safely over an unreliable network.**

Leslie Lamport's Paxos is the foundation that most later protocols build on. Its single-decree form decides exactly one value. It uses three roles, which a real system usually combines in each replica:

- **Proposers** suggest values.
- **Acceptors** vote on values and store their votes durably.
- **Learners** find out which value was decided.

Every proposal carries a **ballot number**. Ballot numbers are totally ordered and unique to each proposer, for example a counter paired with a node id. The ballot number is how Paxos tells new attempts from old ones that the network delivered late. It is the same idea as the term we attached to log entries.

Paxos runs in two phases:

- **Phase 1 (prepare and promise).** A proposer picks a ballot b and asks a majority of acceptors to promise not to accept any lower ballot. Each acceptor also reports the highest-ballot value it has already accepted, if any. If some value may already be decided, the proposer must propose that value instead of its own.
- **Phase 2 (accept).** The proposer sends accept(b, v) to the acceptors. An acceptor accepts unless it has already promised a higher ballot. Once a majority has accepted the same (b, v), v is chosen.

Phase 2 is the quorum replication we described above. Phase 1 is the leadership step. It establishes the right to propose and discovers what earlier leaders may have decided, which is exactly the "hard part" from the replication section. We will spend the next article on it.

What matters here is how Paxos handles the unreliable channel:

- **Lost messages** are fine. A proposer that does not hear from a majority simply retries, possibly with a higher ballot.
- **Duplicated messages** are fine. Accepting the same (b, v) twice changes nothing.
- **Late messages** are fine. An accept from an old ballot arrives at acceptors that have already promised a higher one, and they reject it.

Ballot numbers and quorum intersection together make delay, loss, and duplication harmless to safety. That is the whole trick. Under crash-recovery, it holds only because acceptors persist their promises and accepted values before replying, which is the "remember what you acknowledged" rule from the previous section.

## From one value to a log: Multi-Paxos

**Running full Paxos for every log position is correct but wasteful. A stable leader can skip phase 1.**

A log needs one consensus decision per position. Running both phases for each position doubles the round trips. Multi-Paxos optimizes this: once a proposer completes phase 1 for a ballot, the promise covers all future log positions, not just one. The proposer becomes a *leader*, and for every new entry it only runs phase 2. That is a single round trip to a majority, which is exactly the replication path described in the previous article.

The leader keeps this privilege until another proposer runs phase 1 with a higher ballot. At that point, acceptors reject the old leader's accepts, and the new leader takes over. Leadership in Multi-Paxos is not a separate mechanism. It falls out of ballot ordering.

Lamport's original descriptions left many engineering details open, such as how to handle gaps in the log, how replicas catch up, and how to change membership. Different implementations answered them differently, which is a large part of why Paxos earned its reputation for being hard to implement.

## Raft: replication with an explicit structure

**Raft keeps the same quorum core but gives the log a strict shape, so that replicas can be repaired with simple local checks.**

Diego Ongaro and John Ousterhout designed Raft to be understandable. It decomposes consensus into leader election, log replication, and safety. Its *term* plays the role of the Paxos ballot, and every entry in a Raft log stores the term of the leader that created it, just as we proposed when identifying entries.

### AppendEntries and the consistency check

The leader replicates with a single message, AppendEntries. Besides the new entries, it carries the index and term of the entry immediately before them. The follower accepts the message only if its own log has an entry at that index with that term. Otherwise it rejects, and the leader retries with an earlier position until the logs match. When they do, the follower deletes anything after that point that conflicts with the leader and appends the leader's entries.

This check gives Raft its **Log Matching Property**:

- If two logs contain an entry with the same index and term, the entries are identical.
- If two logs contain an entry with the same index and term, the logs are identical in all preceding entries.

The first follows because a leader creates at most one entry per index in a term. The second follows by induction from the consistency check. Together they mean a follower never needs to compare its whole log with the leader's. A single (index, term) pair is enough.

This is also how Raft survives the unreliable channel. A duplicated AppendEntries is applied idempotently. A reordered one either fits the log or is rejected. A lost one is resent. The follower never has to trust message order, only the content of each message.

### Repairing followers

When a follower is behind or has diverged, the leader walks backward from its own log until it finds the last matching entry, then overwrites everything after it. The leader never deletes or rewrites its own log. Its log is the source of truth for its term.

Followers that are too far behind, or whose needed entries were compacted, receive a snapshot instead, as described in the previous article. The snapshot carries the index and term of the last included entry, so the consistency check continues to work after it.

## The commit rule and why terms matter

**"A majority has it" is not enough. A majority must have it in the leader's current term.**

It is tempting to define commit as "stored on a majority." Raft shows why this is wrong.

<!-- REVIEW: check this walkthrough against Figure 8 of the Raft paper (terms, indexes, which replica wins). -->

The Raft paper describes a scenario with five replicas. A leader in term 2 writes an entry at index 2 and replicates it to one follower before crashing. A new leader in term 3 is elected without that entry and writes a different entry at index 2, then crashes too. The original leader returns, wins term 4, and continues replicating its term-2 entry until a majority stores it. If the leader treats the entry as committed at this point and then crashes, the replica holding the term-3 entry can still win the next election. Its last entry has a higher term, which makes its log look more up to date. It would then overwrite index 2 on every replica, erasing an entry that was supposedly committed.

The fix is Raft's commit rule. A leader only counts replicas for entries from its own current term. Older entries become committed indirectly, once an entry from the current term after them reaches a majority. By the Log Matching Property, every entry before a committed one is committed too.

This is the reason a replication protocol cannot be discussed entirely without leadership. Commitment depends on the term, and terms come from elections. We do not need to explain how a term is obtained yet. We only need two facts: every entry is stamped with a term, and only one leader exists per term.

## Flexible quorums: two phases, two quorums

**The election quorum and the replication quorum only have to intersect each other.**

Paxos and Raft use majorities for both phases. In 2016, Heidi Howard, Dahlia Malkhi, and Alexander Spiegelman showed with *Flexible Paxos* that this is stronger than necessary. The requirement is narrower:

- Every phase 1 quorum must intersect every phase 2 quorum.
- Two phase 2 quorums do not need to intersect each other.

The reasoning follows from what each quorum protects. A phase 2 quorum makes an entry durable. A phase 1 quorum discovers what earlier leaders may have made durable. They must overlap so the new leader cannot miss a committed entry. Two phase 2 quorums never compete, because within a single ballot or term only one leader is replicating.

This means a system can trade one phase against the other. With five replicas, a write could commit on any two replicas, as long as an election contacts four.

<!-- REVIEW: check that the n=5, |Q2|=2, |Q1|=4 example satisfies |Q1| + |Q2| > n. It does (6 > 5). --> Writes get faster and cheaper, and elections, which are rare, get more demanding.

Flexible quorums are a theoretical hook for the design in this series. Replication and election are two distinct problems with two distinct quorums. Raft merges them into a single protocol run by the replicas themselves. Other systems separate them:

<!-- REVIEW: check PacificA's description (primary waits for all replicas in the configuration) and Kafka's (ISR plus controller). -->

- **PacificA**, from Microsoft Research, gives membership and leadership to a separate configuration manager. The primary replicates to every replica in the current configuration.
- **Kafka** replicates to an in-sync replica set and relies on a controller to choose leaders from it.
- The **Coordinator** we introduced in the previous article follows the same separation. The replicas handle phase 2, and the Coordinator is responsible for leadership.

The safety argument is the same in each case. Whoever installs a new leader must observe enough replicas to intersect every possible replication quorum of earlier terms.

## Towards leader election

**Everything in this article was safe because we assumed a legitimate leader. The next article removes that assumption.**

We started from a channel that loses, duplicates, delays, and reorders messages, and from processes whose crashes look like slowness. On top of it, we built a replication path where quorum intersection makes committed entries durable, terms and the consistency check let followers reject stale messages and repair divergent logs, and the commit rule prevents an older leader's entries from being declared committed too early. We also fixed the failure model: crash-recovery with honest storage, not Byzantine.

All of it depends on one unexplained step: how a leader gets a term, and how the system makes sure no older leader keeps acting. In an asynchronous system, a leader that was partitioned away, or paused for thirty seconds, does not know it has been replaced. It may continue serving reads or sending appends.

In the next article, we will cover:

- Paxos phase 1 as an election, and how higher ballots fence older ones.
- Raft's election: randomized timeouts, votes, and the up-to-date log restriction.
- Electing through an external coordinator: fencing a quorum before naming a leader, and truncating divergent logs.
- Stale leaders and reads: leases, clocks, and read-index.
- What a leader change means for sessions, ephemeral entries, and notifications.

## References

<!-- REVIEW: check the reference list (Gray 1978 for Two Generals; the original is Akkoyunlu et al. 1975, consider citing it). -->

- Michel Raynal, *Fault-Tolerant Message-Passing Distributed Systems: An Algorithmic Approach*, Springer, 2018.
- Michael J. Fischer, Nancy A. Lynch, Michael S. Paterson, "Impossibility of Distributed Consensus with One Faulty Process," *Journal of the ACM*, 1985.
- Marcos K. Aguilera, Wei Chen, Sam Toueg, "Heartbeat: A Timeout-Free Failure Detector for Quiescent Reliable Communication," WDAG, 1997.
- Jim Gray, "Notes on Data Base Operating Systems," 1978 (the Two Generals problem).
- Leslie Lamport, "The Part-Time Parliament," *ACM TOCS*, 1998, and "Paxos Made Simple," 2001.
- Diego Ongaro, John Ousterhout, "In Search of an Understandable Consensus Algorithm," USENIX ATC, 2014.
- Miguel Castro, Barbara Liskov, "Practical Byzantine Fault Tolerance," OSDI, 1999.
- Heidi Howard, Dahlia Malkhi, Alexander Spiegelman, "Flexible Paxos: Quorum Intersection Revisited," OPODIS, 2016.
- Wei Lin, Mao Yang, Lintao Zhang, Lidong Zhou, "PacificA: Replication in Log-Based Distributed Storage Systems," Microsoft Research, 2008.

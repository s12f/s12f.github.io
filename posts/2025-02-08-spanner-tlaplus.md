---
title: Implement and Verify Google Spanner in TLA+
nav_parent: Home
---
# Implement and Verify Google Spanner in TLA+

## Tables of Contents
- TOC
{:toc}

## Spanner

[Spanner](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) is a Google's Globally-Distributed Database,
which inspired many modern distributed databases included [tikv](https://github.com/tikv/tikv), [CockroachDB](https://github.com/cockroachdb/cockroach) etc.
This implementation focuses on the [spanner-osdi2012](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) paper,
current Spanner probably has changed a lot.

The TLA+ implementation, models are provided in [tlads/spanner](https://github.com/s12f/tlads/tree/main/spanner),
which are not fully same as paper mentioned due to some reasons
(e.g. to reduce the state space in TLA+),
but it will provide same guarantees.

This post mainly includes some notes about Spanner and the TLA+ implementation,
I am not going to talk much about Spanner and TLA+,
so it is assumed that you know the basic knowledge of Spanner and TLA+.

## TrueTime

### TrueTime In Paper

The ordering of events is one of the most important part for both transaction and replication layers.
Spanner introduces a novel time API called TrueTime to solve the problem.
TrueTime API includes three methods:

| Method       | Returns                              |
| ---          | ---                                  |
| TT.now()     | TTinterval: [earliest, latest]       |
| TT.after(t)  | true if t has definitely passed      |
| TT.before(t) | true if t has definitely not arrived |

`TT.before` is a dangerous method in real world.
Imagine that you call `TT.before(t)` to check if a timestamp `t` is not arrived,
it returns true,
but due to some unstable reasons (hardware problem, GC etc.),
the absolute time could be actually arrived,
so the checking is somehow pointless.

And it is similar to use `~ TT.after(t)` to check that a timestamp `t` has not yet passed.

So the implementation will:
- Never use `TT.before`.
- Use `TT.now().earliest` or `TT.now().latest` to get a timestamp.
- Use `TT.after` to wait that the absolute time definitely passed.
- Only use `~ TT.after` carefully for some situations[^not_after] without compromising correctness.

### TrueTime In TLA+

The full TLA+ specification is in [TrueTime.tla](https://github.com/s12f/tlads/blob/main/spanner/TrueTime.tla),
I use two variables to implement TrueTime API:
- `ttAbs`: the absolute time
- `ttDrift`: the back/left offset of clock drift

So here is the `TTNow` and `TTAfter` definition:
```
TTNow ≜ [ earliest ↦ ttAbs - ttDrift
        , latest ↦ ttAbs + TTInterval - ttDrift ]

TTAfter(t) ≜ t < TTNow.earliest
```
Here is some examples of `TTNow` results with different clock drifts (`TTInterval = 2`, `ttAbs = 2`):
- `ttDrift = 2 => TTNow = [0, 2]`
- `ttDrift = 1 => TTNow = [1, 3]`
- `ttDrift = 0 => TTNow = [2, 4]`

The clock may drift in any moment, even could drift back:
```
TTDrift ≜
    ∃x ∈ 0 ‥ TTInterval:
        ∧ ttDrift' = x
        ∧ UNCHANGED ⟨ttAbs, ttSi⟩
```

The next action of `ttAbs` is quite tricky due to two reasons:
1. `ttAbs` could not increase unlimited.
2. Spanner need `ttAbs` to advance long enough to commit transactions (the commit wait condition).

So there are two actions could make `ttAbs` increasing:

```
\* driven and limited by external events
TTEventNext ≜
    ∧ ttAbs' = ttAbs + 1
    ∧ UNCHANGED ⟨ttSi, ttDrift⟩

\* driven in TTNext, limited by `ttSi` and `TTMaxSi`
TTSiNext ≜
    ∧ ttSi < TTMaxSi
    ∧ ttSi' = ttSi + 1
    ∧ ttAbs' = ttAbs + 1
    ∧ UNCHANGED ⟨ttDrift⟩
```

## Disjointness

As the paper mentioned:

> Spanner depends on the following monotonicity invariant: within each Paxos group, Spanner assigns timestamps to Paxos writes in monotonically increasing order, even across leaders.

> This invariant is enforced across leaders by making use of the disjointness invariant: a leader must only assign timestamps within the interval of its leader lease.

The leader election and lease management of Paxos group must guarantee the disjointness invariant.
Spanner provide a simple algorithm and proof in the paper,
I implement the algorithm in [DisjointLeases.tla](https://github.com/s12f/tlads/blob/main/spanner/DisjointLeases.tla) with the TrueTime API.

The disjointness invariant in TLA+ is presented as:
```
Disjointness ≜
    ∀i1, i2 ∈ intervals:
        ∧ i1.start < i2.start ⇒ i1.end < i2.start
```

the `intervals` variables is the possible lease intervals of leaders.

## Abstraction of Linearizable Storage

Since the nature of model checking,
if the implementation allows too many states,
the model checker will take immeasurable time to verify the specification.
So I abstract the Paxos replication layer as a linearizable key-value storage,
that makes the implementation of transaction much simpler.

There are many ways to a high-available linearizable key-value storage through replicated state machines (e.g. [Raft](https://raft.github.io/raft.pdf)),
but which is out of topic of this post.

So in TLA+, the participant is:
- A linearizable key-value
- Storing a single key instead of a key range

The Initial `ps` (participants) variable in TLA+:
```
ps = [ p ∈ Key  ↦
        [ data ↦ ⟨⟩
        , lock ↦ None
        , prepared ↦ None
        (* Paxos monotonic timestamp, we don't show the S_max
            constraint here by simply assuming S_max will be big enough,
            because it is a implemetation detail in Paxos replication layer.
        *)
        , lastTs ↦ TTNow.earliest
        ] ]
```


## Transaction

The full specification: [Transaction.tla](https://github.com/s12f/tlads/blob/main/spanner/Transaction.tla).

### States of Read-Write Transactions

Since the read-write and read-only transactions use different implementations,
and the read-write transactions is the main part of the transaction layer.
I leave the implementation of read-only transactions to [Read-Only Transactions](#read-only-transactions),
so what I mention transactions in this section, actually means read-write transactions.

In the paper,
transactions are driven by clients to avoid sending data twice across wide-area links,
the commit point of a transaction is whether the commit log is replicated through Paxos group.
For simplicity, the TLA+ specification is different,
which uses a `txs` to present the states of transactions,
and the states of a transaction are store at the coordinator leader of the transaction.
The commit point of a transaction is determined by `status` state.

The initial state of a transaction:
```
(* tx: transaction name/id
   rw: the definition of tx
 *)
TxInit(tx, rw) ≜
    [ status ↦  "Reading"
    , coLeader ↦ SafeChoose(DOMAIN rw.write) \* Coordinator Leader(Participant)
    , rw ↦ rw
    , read ↦ ⟨⟩
    , startCommitTs↦ None
    , commitTs ↦ None
    ]
```
the full status list:
- `Reading`: the transaction started, is acquiring read lock and reading data.
- `Committing`: the transaction have read and written data, starts committing.
- `CommitWaiting`: the transaction archived the commit point, waits until `commitTs` has passed.
- `Committed`: the transaction is committed safely.
- `Aborted`: the transaction is aborted.

### Locking Strategy

The current implementation only supports strict serializable isolation,
which is the same as the paper,
So reads and writes operations acquire the same mutex-like locks instead of different lock types.

Spanner use wound-wait strategy to avoid deadlock,
which means a high-priority transaction could kill low-priority transactions at
`AvoidDeadlock` action if they didn't archive commit point (`CommitWaiting` or
`Committed`).

### Liveness of Unfinished Transactions

In the paper,
transactions will send heartbeats to keep itself alive.
If timeout of a transaction happens,
other transactions could abort the uncommitted transaction.

The TLA+ specification doesn't implement that,
transactions only could be aborted by deadlock prevention.

The recovery function is implemented as the paper mentioned:
clean locked and prepared keys of unfinished transactions.

### Read-Only Transactions

Read-Only transactions in Spanner is lock-free,
and read is valid if
$$ s_{read} <= t_{safe} = min(t^{Paxos}_{safe}, t^{TM}_{safe}) $$ is true,
$$ t^{Paxos}_{safe} $$ is the timestamp of the highest-applied Paxos write.

When I was writing the TLA+ specification,
I found the biggest problem of the formula is that
$$ t^{Paxos}_{safe} $$ is always too low
if $$ s_{read} $$ is assigned the `TTNow.latest`.
That will cause read-only transactions to wait infinitely
if there are not new writes to update the $$ t^{Paxos}_{safe} $$,
which also breaks the liveness property in TLA+.

If a read-only transaction involves a single Paxos group,
assigning `LastTS()` to $$ s_{read} $$ is a good choice as the paper mentioned,
but which doesn't really work in multiple Paxos groups.

After re-thinking about the formula,
The $$ t_{safe} $$ is introduced to prevent
new writes with commit timestamps less than $$ s_{read} $$ after reading,
which break the guarantee that
the data is always up-to-date to read with $$ s_{read} $$.

The solution is simple,
the $$ s_{read} $$ is still assigned the `TTNow.latest`,
but the reads will wait until the $$ s_{read} $$ passed,
so this is the new formula:

$$ s_{read} <= t_{safe} = min(max(t^{Paxos}_{safe}, TTNow.earliest), t^{TM}_{safe}) $$

And the TLA+ code:

```
CheckTSafe(key, ts) ≜
    IF ps[key].prepared = None
    THEN ts ≤ Max({TTNow.earliest, ps[key].lastTs})
    (* since we re-use the prepared filed in the coordinator leader,
        the ps[key].prepared.ts could be the commit timestamp,
        so ts is not valid if it equals the prepared timestamp.
    *)
    ELSE ts < ps[key].prepared.ts
```

### Fairness and Liveness

The liveness properties of the Spanner:

- All transactions will eventually be committed or aborted.
- All related locks will eventually be cleaned.

The TLA+ code:
```
Done ≜
    ∧ ∀tx ∈ DOMAIN txs: txs[tx].status ∈ { "Committed", "Aborted" }
    ∧ ∀key ∈ Key: ps[key].lock = None
```

Verify liveness properties is more expensive (05min 19s with 8 workers) than safety,
so the `PROPERTIES` in config file is commented by default.

To avoid infinitely shuttering,
a weak fairness for `Next` is necessary.
And there are two actions (`Commit` and `RoTxRead`) waiting
until the absolute time (`ttAbs`) definitely passed,
the clock may drift back and drift forward infinitely.
So the weak fairness is not enough for the two actions,
a strong fairness increasing `ttAbs` is also necessary.

The fairness in TLA+:

```
Fairness ≜
    ∧ WF_⟨txs, ps⟩(TxNext)
    ∧ WF_⟨ttSi, ttAbs, ttDrift⟩(TTNext)
    ∧ SF_⟨ttAbs⟩(TTAdvanceForTxLiveness)
```

### Verification of Strict Serializability

Due to some historical reasons,
there are some different formal models of isolation levels,
so there are also different ways to verify the isolation levels of databases.

Here are some papers and posts to talk about that:
- Papers
    - [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf)
    + [Generalized Isolation Level Definitions](https://pmg.csail.mit.edu/papers/icde00.pdf)
    + [Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed Transactions](https://pmg.csail.mit.edu/papers/adya-phd.pdf)
    + [Seeing is Believing: A Client-Centric Specification of Database Isolation](https://www.cs.cornell.edu/lorenzo/papers/Crooks17Seeing.pdf)
- Posts
    - [Seeing is Believing: A Client-Centric Specification of Database Isolation ](http://muratbuffalo.blogspot.com/2022/06/seeing-is-believing-client-centric.html)
    - [Automated Validation of State-Based Client-Centric Isolation with TLA+ (2021) ](https://muratbuffalo.blogspot.com/2022/07/automated-validation-of-state-based.html)
    - [jepsen: MySQL 8.0.34](https://jepsen.io/analyses/mysql-8.0.34)

I implement a general TLA+ library to check isolation levels using the
[Seeing is Believing: A Client-Centric Specification of Database Isolation](https://www.cs.cornell.edu/lorenzo/papers/Crooks17Seeing.pdf)
method,
the full specification is in [tlads/isolation-models](https://github.com/s12f/tlads/tree/main/isolation-models).
Use the library to verify that Spanner guarantees strict serializability is very simple:

```
SIB ≜ INSTANCE SIB_ISOLATION
StrictSerializability ≜ Done ⇒  SIB!StrictSerializableIsolation(InitValue, mapped_txs)
```

### Test Cases

Since Spanner is complicated,
even in the simpler TLA+ implementation,
the state space is still very big.
So [TxTest.tla](https://github.com/s12f/tlads/blob/main/spanner/TxTest.tla)
only includes some typical transaction cases:
```
TxDefs ≜
    { \* non-conflict
     ⟨ [ read ↦ { "k1" }, write ↦ [ k1 ↦ 1 ] ]
     , [ read ↦ { "k2" }, write ↦ [ k2 ↦ 2 ] ]
     ⟩
     , \* write skew
     ⟨ [ read ↦ { "k1" }, write ↦ [ k1 ↦ 1 ] ]
     , [ read ↦ { "k2" }, write ↦ [ k2 ↦ 2 ] ]
     ⟩
     , \* multiple writes and RO transaction
     ⟨ [ read ↦ { "k1" }, write ↦ [ k1 ↦ 1, k2 ↦ 1 ] ]
     , [ read ↦ { "k2" }, write ↦ [ k1 ↦ 2, k2 ↦ 2 ] ]
     , [ read ↦ { "k1", "k2" }, write ↦ ⟨⟩ ]
     ⟩
     , \* circular reference
     ⟨ [ read ↦ { "k1" }, write ↦ [ k2 ↦ 1 ] ]
     , [ read ↦ { "k2" }, write ↦ [ k3 ↦ 2 ] ]
     , [ read ↦ { "k3" }, write ↦ [ k1 ↦ 3 ] ]
     ⟩
    }
```
You want to verify some transactions that are not included in `TxDefs`,
you can add them into `TxDefs` and verify them by TLC.
If you found any problem,
feel free to create issues or PRs.

----

[^not_after]: Performance, TLA+ helper, linearizable point etc.


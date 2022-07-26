# Dragonfly Journal Design

## Redis replication
Below is a very short overview of how redis replication works. For a deep dive please see
[redis docs](https://redis.io/docs/manual/replication/).

Redis replication employs leader-follower (master-replica) paradigm which provides eventual
consistency between replicas.

The system works using three main mechanisms:

1. When a master and a replica instances are well-connected, the master keeps the replica updated by sending a stream of commands to the replica to replicate the effects on the dataset happening in the master side due to: client writes, keys expired or evicted, any other action changing the master dataset.

2. When the link between the master and the replica breaks, for network issues or
because a timeout is sensed in the master or the replica, the replica reconnects
and attempts to proceed with a partial resynchronization: it means that it will try to just obtain the part of the stream of commands it missed during the disconnection.

3. When a partial resynchronization is not possible, the replica will ask for a full resynchronization. This will involve a more complex process in which the master needs to create a snapshot of all its data, send it to the replica, and then continue sending the stream of commands as the dataset changes.

Redis uses asynchronous replication. When replicas are first connected to their leader,
they trigger scenario (3), they get the full snapshot and then the change log since the point the snapshot was triggerred. This is how a full synchronization works in more details:

The master starts a background saving process to produce an RDB snapshot and to push it into
a replica socket (full-sync phase with diskless replication). At the same time it starts to
buffer into a replication buffer all the mutation commands sent by its clients.
When the background saving is complete, and the snapshot is digested by replica, the master will then
send all the buffered commands to the replica. The last stage called "stable state replication" because
it never ends unless the replica disconnects. This replication stream data uses the same format
as Redis protocol (RESP).

## Problems with Redis replication design
The replication buffer is kept in memory and has a predefined limit that is controlled by a special
configuration directive `client-output-buffer-limit` or COB limit. The replication buffer is implemented
as a ring buffer - once it gets filled it starts overriding its oldest records.

It takes a lot of time to produce an rdb snapshot and then to load it on the replica side (full-sync phase).
In fact, this time is linear with the size of the database, so practically unbounded.

Now, Redis, being a high throughput database, may sustain thousands of writes per second.
It requires lots of RAM to keep all of these changes during the full-sync phase.
A Redis operator may need to reserve lots of memory using `client-output-buffer-limit`
and even this might not suffice. If the leader overrides its records before replica has a chance
to pull them from the replication log, then the replica won't be able to finish the replication
flow and will need to start from the scratch.

This problem is described extensively in [this post](https://redis.com/blog/the-endless-redis-replication-loop-what-why-and-how-to-solve-it/).


To summarize, Redis replication requires reserving a memory buffer in advance. The memory buffer usage is dependent on many factors such as database size, write traffic throughput and replica processing speed.
Even with a very large capacity reservation for the replication buffer, it is not guaranteed
that the replication will succeed.


### Related features

 1. Redis allows writing all its changes into Append-Only-File (AOF).
    AOF is not used for replication, but for enhanced durability. Due to its performance problems, AOF
    is not used widely. Moreover, some managed services providers do not support AOF at all.

 2. Redis allows maintaining multiple replicas per leader. This is especially common for HA setups that reside in multiple AZs. This introduces additional complexity and makes the whole system even less reliable.

## DF replication using journal - high-level sketch

We introduce the concept of a replication journal that keeps all the database changes on disk.
In a way the DF journal is similar to Redis AOF file or a database WAL.

Before writing into a journal, DF batches recent changes in a intermediary ring-buffer in memory.
This helps DF to limit its write latency and to maximize the underlying disk I/O performance.
The replication buffer is constantly offloaded to disk by an asynchronous process. The journal on-disk format is optimized for direct I/O.

Once the leader (master) starts a full-sync with the replica, the leader
creates the point-in-time snapshot that is passed to the replica. It keeps then the transaction id of the snapshot in the replication context. Once the replica finishes its full-sync phase, it pulls the changes gathered so far.
The changes are stored in the journal, so there is no risk of overflowing on memory usage.
The leader sends changes starting from the transaction id after the snapshot. If a journal record is still kept in the replication buffer, the master sends it from there, otherwise it's read from the disk.

Given that the replica can load items faster than the leader produces them, the replication will eventually reach the state where most records are sent to replica directly via the replication buffer.

TBD: flow and state machine diagrams.

Such design allows us to avoid the pitfalls of having memory blowup that Redis has.
It won't cause endless sync loops and will be more much easier to tune and maintain.
DF journal will also have the ability to replace Redis AOF that is designed to provide
better durability guarantees for Redis store. DF journal opens us to possibly more advanced
features like a master-less sync using the RDB snapshot together with the journal log stored in a cloud storage.

## Journal detailed design
The description above is somewhat rough and high level. The sections below cover detail
requirements and the possible solution for DF journal implementation.
We use the term `Producer` for a DF process
that writes into the journal, and `Loader` - to describe a DF process that reads DF journal.
A journal producer can be a DF leader that sends the replication stream to its replica but also
a single DF node that uses journal for enhanced durability.

Similarly, a DF follower is a journal loader, but also a single DF node that loads the journal from the storage.


### Design Requirements

1. Dragonfly is designed on top of shared-nothing, thread-per-core architecture. Our journal should
preserve its high performance qualities. Specifically it should not force inter-thread synchronization, or introduce other significant bottlenecks.

2. DB operations on the loader should be applied in the same order they were applied by the producer.
Moreover, those operations should be determenistic and they should produce the same state as in the producer.

2. DF Journal file format should support seeking. It does not have to be optimized for random access,
but it should allow skipping parts of it without parsing the whole journal from the start. This requirement is based on the fact that we must support fetching a journal suffix based on the transaction id.

3. DF journal should be still parseable if its prefix is erased. Consider the scenario, where the journal fills the disk capacity. We would need to erase its earliest records and still be able to read the rest of it.

4. Redis leader and follower synchronize themselves using the physical (file) offset in the replication log stream. Such protocol is too fragile and ambiguous. DF Journal should maintain log sequence numbers (LSN) that uniquely identify each record. DF leader and follower should not depend on the physical representation of the journal, instead they should synchronize using LSNs.

5. DF journal format should be reasonably balanced between a compressed representation that might
   use lots of CPU and its on-disk size efficiency.

6. DF journal records should be reasonably sized. That means the format should handle "whales" efficiently: i.e. the operations that have very large keys or values, or the operations with many keys or subkeys.

7. A replica reading the journal should be able to apply operations atomically.
Whether it reads the journal stream from a socket or from a file, when an I/O error happens and the stream was interrupted, the replica should be able to stop before appplying operations that have partial data or just some of the keys. That applies, of course, to both multi-key and multi-command transactions.

### Thread actors in DF

DF used thread-per-core architecture and employes fibers for asynchronous processing.
Each client connection in DF is managed by a dedicated fiber and every fiber is pinned to a thread.

DF in-memory database is sharded into `N` parts, where `N` is less or equal to number of threads
in the system. Each database shard is owned and accessed by a single thread.
The same thread can handle tcp-connections and also host a database shard. See the diagram below.

<br>
<img src="http://assets.dragonflydb.io/repo-assets/thread-per-core.svg"
     style="background-color:white;padding:10px;" border="0"/>

Here, our DF process spawns 4 threads, where threads 1-3 handle I/O (i.e. manage client connection),
threads 2-4 manage DB shards. Thread 2, for example, divides its cpu time between handling incoming
requests and processing db operations on the shard it owns.

Due to shared-nothing design, each thread has access to only data it owns. Consider the case,
where a connection pinned to thread 1 handles "MSET" transaction for keys residing on shard 2, 4;
and another connection pinned to thread 2 processes "MSET" transaction for keys residing on shard 4.
The keys of both transactions do no overlap, therefore DF does not perform any type of serialization
between those transactions. The diagram below depicts one of the possible orders of events with DF threads. See how `TH2` thread handles db operation on its shard as part of the execution orchestrated by `TH1`, and then switches to orchestrate `TX2` transaction.

<img src="http://assets.dragonflydb.io/repo-assets/tx-flow.svg"
     style="background-color:white;padding:10px;" border="0"/>

The alternative order of events, where `TH2` would first start orchestrating `TX2` transaction and
then applying changes to `X` is also possible.

### DF journal file parallelism

Unlike with Redis that maintains a single replication buffer or writes changes into a single AOF file,
DF journal is comprised of multiple files. Following shared-nothing philosophy, each DF thread
writes changes in its own file and records in it the events that were handled by that thread.

DF stores there just enough information in those files, so
that the loader could reconstruct the same state as on the producer and to satisfy
the design requirements above.

The naive approach for the example above, would be is that `TH2` write into its
file `J2`:

```
write X <- ...

```

and thread `TH4` would append to `J4`:

```
write Y <- ...
write B <- ...
write C <- ...
```

but then when the replica that applies those logs would not understand that `X,Y`
or `B,C` should be mutated atomically. And if data in `J4` is lost,
there is not enough information to ensure that the replica stops **before** applying changes to `X`.

Therefore, DF journal must also contain meta information about the transaction context
of each operation. `TH1` handles the orchestration of `TX1`, therefore the correct approach
would be:

J1:

```
start TX1, spawns shards 2, 4.
```

J2:

```
TX1:write X <- ..., finish_my_part(TX1).
start TX2, spawns shard 4.
```

J4:

```
TX1:write Y <- ..., finish_my_part(TX1).
TX2:write B <- ...
TX2:write C <- ..., finish_my_part(TX2).
```

The same journal file `J2` constains different kinds of records: a) those that originate from
transaction coordinators and b) those that describe how db executors applied their changes.

Now the loader can see that TX1 is split into 2 parts, verify that each part was fully read from its
owning file (or socket) and only then apply TX1.

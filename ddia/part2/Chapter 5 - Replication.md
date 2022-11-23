# Replication


## Leaders and Followers - Single Leader Replication

The single leader replication model relies on one node to process all incoming writes (the 
*leader*), with all remaining nodes (the *followers*) replicating the leader's data. Both the 
leader and any followers can serve any read request (assuming no partitioning for now), but only 
the leader can process writes. 

The mechanism for how changes are propagated from the leader to the follower is through a 
replication log that tracks all updates processed by the leader. Followers use the replication 
log to update their respective local copies. Followers maintain their own copies of the 
replication log to keep track of which writes have been copied, which is useful when 
recovering from node outages.


### Synchronous vs Asynchronous Replication

A follower can replicate the leader *synchronously* or *asynchronously*:

- **Synchronous replication**: Leader waits until follower has confirmed replication before 
  reporting the update as successful
- **Asynchronous replication**: Leader sends change to follower via replication log / event 
  stream but does not wait for confirmation to report the update as successful

Synchronous replication maintains data consistency at the expense of write availability, as 
writes are not possible if a synchronous follower is unavailable. In the extreme case where all 
followers are synchronous, a failure in *any* node revokes write availability. In practice, a 
system with synchronous replication only marks one follower as synchronous, with the rest of the 
followers remaining asynchronous (i.e. a *semi-synchronous* system).

Asynchronous replication improves write availability (writes can be processed even if all 
followers fail) at the cost of data consistency (i.e. data reads across nodes at a given point in 
time might not yield the same results). In addition, any unreplicated writes not persisted to the 
replication log will be lost in case the leader fails (i.e. weakened durability). However, in 
practice asynchronous replication is still a popular choice.


### Setting Up New Followers

New followers can be set up in a process similar to replication in existing followers:

- Take a consistent point-in-time snapshot of the leader's database and link that snapshot to 
  the appropriate position in the leader replication log
- Copy the snapshot to the new follower node
- Connect the follower to the leader and process all updates in the replication log post-snapshot


### Handling Node Outages

If a follower node fails, it can recover by comparing its replication log to that of the leader 
and process all incremental requests (much like how certain storage engines rely on a 
write-ahead log to rebuild an index). This process is known as *catch-up recovery*.

The high-level process when a leader fails (known as *failover*) is typically as follows:

- The database recognizes that the leader node has failed. Most systems detect that a 
  node has failed if it is unable to respond to regular system health pings.
- The database elects a new leader node, typically the node with the most up-to-date changes 
  from the original leader's replication log.
- The database reconfigures the system to direct write requests to the newly minted leader node. 
  Follower nodes use the new leader to replicate writes.

Note that the failover process contains challenges:

- Writes on the old failing leader that have not been replicated might be lost, particularly if 
  the new leader receives conflicting writes as the old leader spins back up
- Two nodes might simultaneously believe they are the leader, which could lead to conflicting writes
- The time threshold to determine whether a leader is dead needs to be properly calibrated - 
  waiting too long can hurt availability, while not waiting long enough can lead to unnecessary 
  failovers


### Implementation of Replication Logs

There are a few options when implementing the replication log mechanism to propagate 
database changes to follower nodes:

- *Statement-based replication*: Statement-based replication attempts to execute each write 
  statement in follower nodes. This approach runs into challenges when encountering 
  non-deterministic functions (e.g. `NOW()`, `RAND()`), environment-specific side effects, 
  and concurrent transaction ordering.
- *Write-ahead log shipping*: The write-ahead log on the leader can be distributed amongst 
  followers upon database update across the network. The drawback of this approach is that the 
  representation of the WAL is very low-level and is tightly coupled with the underlying storage 
  engine implementation and version. Rolling upgrades where certain nodes are upgraded without 
  downtime are risky as a result, since the WAL from a different version might not be compatible.
- *Logical / row-based log replication*: The logical log replication strategy is similar to WAL 
  shipping, except the logical log is an easily consumable abstraction over the WAL that is not 
  coupled with storage engine implementation details. It does not suffer from the drawbacks of 
  statement-based replication (since only the transaction results are serialized to the log, and 
  not the queries themselves) or WAL shipping (since the abstracted representation is decoupled 
  from storage details).
- *Trigger-based replication / stored procedures*: In this strategy, the application developer 
  defines a custom piece of business logic that is triggered in response to each write. This is 
  useful if custom event-driven logic for data replication is necessary.



## Problems with Replication Lag

As mentioned above, synchronous replication can be prohibitively expensive and fragile to 
implement in a distributed system as the number of nodes increases (in short, because relying on 
synchronous replication across all nodes blocks writes if *any* node goes down). The consequence 
of relying on asynchronous replication in a distributed system is the phenomenon of *eventual 
consistency*, where followers will eventually catch up to and become consistent with the leader. 
The time required to achieve consistency is known as the *replication lag*. 

Under normal operations, replication lag may last a fraction of a second and might not be 
noticeable. However, in scenarios with high load or network problems, this lag might extend to 
several seconds / minutes. The below covers some issues arising from replication lag:

- **User submits data that is "missing"**: A user submits some data to write to the leader, then 
  immediately attempts to read the submitted data. If the follower serving the read has not 
  replicated the write, then the user will see that the submission is "missing". Avoiding this 
  situation requires *read-after-write consistency*, which can be implemented by intelligently 
  routing reads to specific nodes (either the leader or to any follower that has replicated at 
  least until the user-submitted write). This can be complicated if several data centers serve 
  read requests at the same time, and/or if the user accesses application data across multiple 
  devices (since any client-side state tracking last write time cannot be shared across devices).
- **User views data that "moves backwards in time"**: A user reads data twice, where the first 
  request is served by a follower that is consistent with the leader and the second request is 
  served by a follower that is behind. In this case, the user might see data revert to a prior, 
  stale state. Avoiding this situation requires *monotonic read consistency*, which means that 
  if one user makes several reads in sequence, a later read will serve data at least as fresh as 
  prior reads. This can be achieved by ensuring that each user's reads are always served by the 
  same node, but this runs into a challenge if that node fails. 
- **User views data that "violates causality"**: A user reads two pieces of data where there is 
  a causal dependency (e.g. a question and an answer in a chat application), but the result is 
  served before the cause (e.g. the answer is served before the question in the chat application)
  . Avoiding this situation requires a *consistent prefix reads* guarantee, which means that 
  reads will serve writes in the sequence they occur. This can be achieved by assigning a 
  global ordering to writes, or by ensuring that any causally related writes are written to the 
  same partition at the same time, but these options are not always feasible.

None of these solutions is particularly easy to manage in application code. *Transactions* exist 
to allow the database to offer stronger guarantees, instead of delegating to the application.



## Multi-Leader Replication


### Use Cases for Multi-Leader Replication

Situations where a multi-leader configuration may be appropriate include:

- **Multi-datacenter operation**: An application with replicas in several different datacenters 
  / regions can benefit from a multi-leader configuration, since it would provide better 
  performance (requests do not need to be routed to a specific datacenter with the leader as in 
  a single-leader configuration, but rather can be routed to the closest datacenter with a 
  leader) and robustness to network issues (inter-datacenter replication is typically performed 
  synchronously in a single-leader configuration, but can be handled asynchronously in a 
  multi-leader configuration).
- **Clients with offline operation**: An application that supports offline operation effectively 
  is a multi-leader configuration, since an offline device acts as a leader (i.e. it accepts 
  writes) along with the server.
- **Collaborative editing**: A collaborative editing application (e.g. Google Docs) is similar to 
  the case above, where the client's local state (e.g. the document on the browser) acts as a 
  leader along with the server.

In all of these situations, there is a risk of write conflicts that needs to be addressed.


### Handling Write Conflicts

#### When does a conflict occur?

A conflict occurs when two leaders independently accept inconsistent writes on the same piece of 
data. This conflict is detected when the two leaders then attempt to asynchronously replicate 
their respective changes across other nodes. Note that while synchronous replication can be used 
to avoid write conflicts, it eliminates the main advantage of using multi-leader replication in 
the first place: allowing each replica to accept writes independently.

#### Conflict avoidance

One approach to resolving conflicts is to try and prevent them from occurring in the first place.
For example, an application can route requests from a given user to the same datacenter, to 
ensure that there is only one source of truth per user. However, this is not a permanent 
solution, since traffic would need to be rerouted to a different datacenter in case of failure - 
the case of conflicting writes across leaders still needs to be addressed.

#### Converging toward a consistent state

Some conflict resolution strategies include:

- **Last write wins**: Assign each write a unique ID that can be ordered (e.g. UUID, timestamp, 
  random number) and declare the write with the largest ID among conflicting writes to be the 
  "winner". Conflicting data is discarded. This approach obviously suffers from the risk of data 
  loss, and a possibly bad user experience.
- **Replica precedence**: Assign each replica a unique ID, and declare writes originating from 
  the replica with the largest ID among conflicting writes to be the "winner". Conflicting data 
  is discarded. This approach suffers from the same limitations as above.
- **Value merging**: Define merging logic to merge the values of all conflicting writes (e.g. 
  concatenation).
- **Custom conflict resolution logic**: Record conflicts in an explicit data structure that 
  preserves all information, and write application code that resolves the conflict in some 
  well-defined manner (e.g. by prompting the user).


### Multi-Leader Replication Topologies

A *replication topology* defines the path along which writes are propagated from one node to 
another. Some possible structures include:

- **Circular topology**: Each node receives writes from one node and forwards those writes (plus 
  any writes of its own) to one other node. The drawback with this structure is that a failure 
  in one node can interrupt the flow of write propagation to the rest of the system.
- **Star topology**: One designated root node forwards writes to all of the other nodes (this 
  can be generalized to a tree). The drawback with this structure is that a failure 
  in one node can interrupt the flow of write propagation to the rest of the system.
- **All-to-all topology**: Each node forwards writes to every other node. While this approach is 
  more fault-tolerant to node failures, it can also lead to more variance in replication across 
  nodes as certain edges can have faster network connections than others.

Note that a single-leader configuration is just a special case of the general multi-leader case, 
in which no writes are propagated back into the leader node.



## Leaderless Replication

In a leaderless replication architecture, all nodes in a cluster can accept writes and serve 
reads. Key real-life examples of systems that support leaderless replication include Amazon's 
Dynamo and Cassandra.


### Writing to the Database When a Node Is Down

Rather than rely on specific nodes to manage writes, a leaderless database uses *quorums* across 
nodes to determine whether a write is successful. In other words, the database:

- Sends out a given write request to several (usually all) nodes in the database cluster
- Considers the write as successful as soon as *w* nodes report that the write was processed, 
  where *w* represents a predefined quorum write value

In this manner, the database remains available for writes despite node failures, as long as the 
cluster has enough nodes to reach a quorum. Each write is also assigned a *version number* to 
order writes in the case of stale reads / write conflicts.

Reads follow a similar principle. At read time, the database:

- Sends out a given read request to several (usually all) nodes in the database cluster
- Collects the first *r* responses for the read request, where *r* represents a predefined 
  quorum read value - note that the responses might be different across nodes if the 
  corresponding write(s) did not fully propagate across the queried nodes
- Returns the response with the latest version number

Because quorum writes do not guarantee that all nodes in a cluster will contain the same value, 
the database needs some mechanism to ensure that all nodes converge to a consistent state. 
Dynamo-style datastores often use two mechanisms:

- **Read repair**: At read time, the database detects that a particular node returned a stale 
  response (i.e. it returned a result set with an outdated version number compared to other 
  nodes in the quorum read). The database automatically updates that node with the more up-to-date 
  value. This approach works well for values that are frequently read, but can let data get 
  arbitrarily stale if it is not read (which in turn reduces the overall durability of the 
  system, as restoring from a replica with stale data will effectively delete writes).
- **Anti-entropy process**: The database maintains a background process that constantly looks 
  for stale data across replicas and updates accordingly. This process helps overcome the 
  limitations of the read repair method. Note that unlike the replication log 
  in leader-based replication, this process does not copy writes in any particular order. There 
  may be a significant delay before data is copied.


#### Quorums for reading and writing

Define `n`, `w`, and `r` as follows:

- `n` represents the number of nodes / replicas in the database cluster
- `w`represents the number of nodes needed to achieve a write quorum
- `r` represents the number of nodes needed to achieve a read quorum

The following holds true about the database:

- If `w < n`, writes can still be processed if a node is unavailable 
- If `r < n`, reads can still be processed if a node is unavailable
- If `w + r > n`, then reads will return an up-to-date value (there is at least one node at read 
  time that wrote the latest value at write time)
- Higher values of `w` make writes more susceptible to failure (as fewer node failures are 
  needed before a quorum cannot be satisfied) but allow for lower values of `r` without 
  violating the above condition, which can speed up reads. As a result, systems with few writes 
  and many reads benefit from high values of `w`. Conversely, systems with a relatively high 
  write load benefit from higher `r` and lower `w`.
- Setting `w + r <= n` allows for lower latency and higher availability, but risks reading stale 
  values (the further away from `n`, the greater the likelihood of reading stale data).

A common choice for quorum settings is to make `n` an odd number and set `w = r = (n + 1) / 2`.


#### Limitations of Quorum Consistency

While `w + r > n` is a theoretically sufficient condition for eliminating the risk of reading 
stale data, in practice there are a number of limitations with quorum consistency:

- Certain leaderless applications support *sloppy quorums*, which are a mechanism to help 
  improve availability. A sloppy quorum is a mechanism by which nodes in the database cluster that 
  are not in the designated `n` quorum nodes accept writes on behalf of the quorum nodes (e.g. 
  in the case where the client is unable to connect to the quorum nodes directly). These nodes 
  then propagate the writes back once the network issue has been resolved (a process known as 
  *hinted handoff*). While this approach improves availability (clients unable to connect to 
  quorum nodes are still able to submit writes), it violates the quorum condition above, since 
  it's possible that none of the `r` quorum nodes queried at read time have the latest value to 
  external nodes. Put simply, sloppy quorums improve availability and durability at the expense 
  of read consistency. 
- Concurrent writes face the same challenges as mentioned above - because leaderless replication 
  is an extreme case of multi-leader replication, the issue of conflict detection and resolution 
  still needs to be handled in some manner.
- A write and read executing concurrently may or may not return the concurrently written data in 
  the read.
- A write may succeed on some nodes while failing in others, in which case it is not rolled back 
  on the replicas where the write succeeded. In this case, subsequent reads might read the 
  partially written value.
- A node that fails and is restored from a prior state might contain stale data, which risks the 
  possibility that the number of replicas with the latest value falls below `w`.

For these reasons, it's better to avoid treating leaderless replication as a system that 
guarantees consistency, despite what the quorum condition might imply.
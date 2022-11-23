# Partitioning


## Partitioning and Replication

Datasets that are very large and/or have very high query throughput need to be *partitioned* or 
*sharded* across several nodes. A *partition* (also known as a *shard*) represents a piece of a 
broader dataset - typically, a given piece of data belongs to one and only one partition. A 
partitioned dataset is managed by *nodes*, where each node represents an individual machine in a 
cluster. A given node can have multiple partitions.

The primary reason why datasets are partitioned is scalability:

- Partitioning a dataset allows a system to take advantage of horizontal scaling via commodity 
  hardware (as opposed to the alternative of having to vertically scale up with increasingly 
  expensive hardware)
- Query load can be parallelized across many nodes / processors (assuming queries can be handled 
  by the same node)

Partitioning is often combined with replication to improve fault tolerance, which means that while
each record may belong to only one partition, that record may still show up across several nodes as
part of partition replication. In the general case, one node will have multiple partitions and can
act as a replication leader for some of its partitions but as a follower for the remaining.


## Partitioning of Key-Value Data

Any system that partitions data needs to figure out how to:

- Define the number of total partitions and assign records to each partition
- Assign partitions to nodes

The objective is to ensure that the end result leads to evenly distributed data and query 
load across partitions / nodes. Uneven partitions can reduce the effectiveness of partitioning, 
as an uneven data / query load can lead to some nodes being overworked while others remain idle. 
Datasets that are unevenly partitioned are said to be *skewed*, and the disproportionately 
large partitions are known as *hot spots*.

### Partitioning by Key Range

In this scheme, each partition is responsible for a certain continuous range of keys (it is 
assumed that keys can be ordered in some meaningful way) - a given record is then assigned to 
the appropriate partition based on the range to which its key belongs. Key points:

- The records within each partition can be persisted in sorted order (e.g. via the SSTable 
  strategy), which means that range queries can be performed efficiently. A range query can be 
  executed simply by identifying the partition(s) containing the specified range and returning all 
  data within the range bounds (if the node(s) containing the partitions are also known, then 
  those nodes can be queried directly - otherwise, the query can be routed through the database 
  to the appropriate nodes).
- The ranges of keys managed by each partition may not necessarily be evenly spaced, since the 
  underlying data may not be evenly distributed
- Certain access patterns may lead to hot spots (e.g. if the partition key is day, then all 
  traffic generated within a day will write to the same partition while others remain idle)

### Partitioning by Key Hash

In this scheme, each partition is responsible for a certain continuous range of key *hashes* - a 
given record is then assigned to the appropriate partition based on the range to which its 
hashed primary key belongs. Key points:

- A good hash function creates a uniform distribution of hash values for a key range, which 
  means that hash partitioning can facilitate a more even data and query load across partitions
- Range queries cannot be performed efficiently as similar keys are scattered across different 
  partitions - since there is no sort order on the keys themselves, returning data for a range 
  of keys becomes more expensive (e.g. the database may need to query all partitions for a given 
  range)

### Partitioning by Key Hash and Range

Cassandra achieves a compromise between range and hash partitioning by allowing a declaration of 
a multi-column compound primary key. The first column value for a given record dictates the 
partition via key hash (i.e. hash partitioning), and subsequent columns are used to maintain 
sort order among records within the partition via SSTable. This hybrid allows for efficient 
range querying given a fixed value for the first column.

### Skewed Workloads and Relieving Hot Spots

Key hash partitioning helps reduce partition skew when *a range* of keys are driving a 
disproportionately large volume of traffic (since those keys would have otherwise been mapped to 
the same range and therefore the same partition), but it does not help if a *single* key is 
generating the traffic (since the same key will have the same hash, and therefore map to the 
same partition). Such conditions can be common in the wild (e.g. several reads of and responses 
to a celebrity's post on a social media site).

One way of addressing partition data skew in such situations is to add a randomly generated 
value (i.e. *salting* the key) to encountered instances of the hot key (at the beginning for range 
partitioning, either beginning or end for hash partitioning). This helps split the key across 
multiple partitions (e.g. adding a random two digit number maps the original key writes evenly 
across 100 new keys, which can be distributed to different partitions). However, this approach 
introduces additional overhead:

- The system needs to keep track of which original keys were split in this manner
- Reading records associated with the original key requires querying all partitions for all 
  possible salted key values

For this reason, it's better to salt hot keys only when necessary.


## Partitioning and Secondary Indexes

If records are only ever accessed by their primary keys, then requests can be routed to the 
appropriate node(s) / partition(s) under either a range or hash partitioning scheme. However, if 
records are accessed by secondary keys, then building secondary indexes on partitioned data may 
be necessary.

### Document-Partitioned Secondary Indexing (Local Indexing)

In this secondary partitioning scheme, each partition indexes its own data on the 
declared secondary key independently - it is not affected by data / indexing in any other 
partition. This partition-scoped secondary index maps each encountered value of the secondary 
key to the corresponding primary key record(s) with that secondary key value in the given 
partition. Key points:

- **Writes are cheap**: Writing a record to the database simply requires updating the secondary 
  index in the corresponding partition - other partitions and their corresponding local 
  secondary indexes remain unaffected.
- **Reads can be expensive**: Reading records from the database by secondary key requires querying 
  all partitions and combining the results, in an approach known as *scatter / gather*. This 
  approach is prone to tail latency amplification (e.g. if one node is lagging). It is still 
  commonly used by many database systems such as MongoDB and Cassandra.

### Term-Partitioned Secondary Indexing (Global Indexing)

In this secondary partitioning scheme, all partitions share a global index that itself is 
partitioned across nodes (grouping the global index in one node would require all secondary 
queries to go through that node, effectively guaranteeing uneven query distribution and 
defeating the purpose of data partitioning in the first place). Different nodes maintain 
different pieces of the global index, so that certain values of a secondary key can be 
maintained by one node while other values can be maintained by others. The partitioning scheme 
for the secondary index itself does not need to match the primary key partitioning scheme. Key 
points:

- **Writes are expensive**: Writing a record to the database might require updating several 
  nodes if multiple secondary indexes are used. As an example, the incoming record `R` may be 
  defined by primary key `a` and secondary key values `b` and `c`. If the secondary index for 
  all records with value `b` in the first secondary key is maintained by node `n1`, and the 
  secondary index for all records with value `c` in the second secondary key is maintained by 
  node `n2`, then both `n1` and `n2` will need to update their corresponding secondary indexes 
  with the value `a`. Note that in a document-partitioned local approach, all secondary indexes 
  would simply be maintained by the partition / node to which `p` was written.
- **Reads are cheap**: Reading a record based on a secondary value only requires querying the 
  node storing the piece of the global index with the given value. The global index will 
  indicate which partition(s) contain the relevant primary key record(s) (and if clustered 
  secondary indexes are possible, it might also return the records themselves directly). 
- **Reads may not be consistent**: Updates to global secondary indexes are often asynchronous, 
  meaning that a write might not be immediately reflected in the global index.


## Rebalancing Partitions

Partitions need to be rebalanced across nodes whenever the set of nodes changes in some way (e.g.
if a new node is introduced to handle increased query throughput / data size, or if a node goes 
down for some reason). The rebalancing problem can be framed as follows:

- After rebalancing, the load (data storage, read / write requests) should be fairly balanced 
  across nodes
- The database should accept read and write requests during rebalancing
- The amount of data transferred across nodes for rebalancing should be minimized

### Bad Strategy: Hash key modulo number of nodes *N*

The natural approach of mapping a key to a node is to simply calculate the hash of the key 
modulo *N*, where *N* represents the number of nodes in the cluster (naturally, this strategy 
would only be applicable to hash partitioning, as keys within the same range are not guaranteed 
to end up in the same node). 

However, this approach fails because any change to *N* (e.g. a new node is added or an existing 
node fails) requires moving *all* the data from their original node to a new one based on the 
updated modulo calculation. This rebalancing approach can be very inefficient, given the potential 
frequency of adding and removing nodes in a cluster. 

### Strategy 1: Fixed number of small partitions

This approach consists of defining a fixed number of partitions to store a dataset. Whenever a 
new node is introduced to a cluster, it can absorb partitions from each of the remaining nodes 
to ensure the resulting distribution is fair (the converse operation occurs when a node is 
dropped). Key points:

- **Operationally simple**: Because the number of partitions is fixed, the underlying 
  database can be simpler to create and manage.
- **Many small partitions**: Because the number of partitions is fixed, and because rebalancing 
  occurs at the partition level, it is common to create many small partitions at database setup. 
  This helps hedge against a growing dataset size in the future (creating too few partitions can 
  lead to fat partitions) and ensures that rebalancing can occur at a reasonably granular level 
  (as fat partitions may not lead to even data storage load). 
- **Not suited for highly variable dataset sizes**: Creating too few partitions at database 
  setup means that performance can degrade for larger datasets (since fat partitions may 
  lead to expensive rebalancing and uneven node activity), but creating too many can degrade 
  performance for smaller datasets (since partitions have their own overhead to manage). 
  Choosing the "right" number of fixed partitions for a variable dataset might not be possible. 

### Strategy 2: Dynamic number of partitions

This approach consists of defining an initial number of partitions to store a dataset, and then 
letting the database manage the number of partitions via dynamic splitting and merging. Whenever 
a partition exceeds a preconfigured size, the database splits it into two partitions that can 
then be sent to two different nodes if necessary to distribute the load (a similar process 
occurs when a partition shrinks below a preconfigured size). Key points:

- **Operationally complex**: Dynamic partitioning can add more complexity for the database 
  (though it's not obvious how much additional complexity the end user / administrator of the 
  database generally faces).
- **Requires pre-splitting**: Because the number of partitions is a function of overall dataset 
  size, there is no information on the "correct" number of partitions when storing the dataset 
  initially. In the extreme case, when the dataset is smaller than the preconfigured partition 
  size, all query traffic might be directed to the same node while the rest remain idle. To 
  avoid this situation, databases with dynamic partitioning enable setting a pre-configured 
  number of partitions to initially manage the dataset (known as *pre-splitting*).

### Strategy 3: Fixed number of partitions per node (consistent hashing)

This approach consists of defining a fixed number of partitions *per node* (as opposed to the 
above strategy, which scales off of the dataset size). When a new node joins the cluster, it is 
assigned a fixed number of partitions, each of which is placed randomly on an abstraction known 
as a *hash ring* (i.e. a continuous ring covering the full range of the given hash function). 
Each key in the dataset can be mapped to a specific location in the hash ring via a hash function, 
which in turn can be mapped to a partition (e.g. each key is assigned to the partition closest 
to it on the ring while traversing clockwise / counterclockwise). Rebalancing occurs only for 
those keys that would be mapped to new partitions from the new node, without affecting any other 
key mappings. This technique is referred to as *consistent hashing*. Key points:
 
- **Only applicable for hash partitioning**: The hash ring abstraction needs to know the full 
  range of values *a priori*, which is not always possible for the key values themselves (defining 
  an unbounded range like Z+ might lead to uncontrolled partition / data skew). A hash function 
  can map key values to a known range of values.
- **Solves the hash modulo *N* problem**: In the (bad) hash modulo approach defined above, *all*
  keys are remapped to a new node whenever a new node is introduced to the cluster. In this
  partitioning scheme, only the keys that are closer to the new partition(s) on the hash ring
  are remapped.
- **Relies on probabilistic balancing**: Because partitions are randomly assigned to locations 
  on the hash ring, it is possible that certain partitions might cover larger hash ranges than 
  others, which can lead to data skew. However, as the number of partitions on the hash ring 
  increase, the mean range covered by each partition should converge to even distribution.

### Operations: Automatic or Manual Rebalancing

- **Automatic rebalancing**: Convenient, but can lead to unpredictable behavior / overly 
  aggressive rebalancing, which can affect overall query performance. In the extreme case, an 
  overloaded node might result in a triggered rebalance that further overloads the node (since 
  data needs to be moved over from it) and potentially other nodes, which triggers additional 
  rebalancing. The end result might be a cascading failure of the cluster.
- **Manual rebalancing**: More predictable, but requires high-quality cluster telemetry and 
  active human-in-the-loop effort.


## Request Routing

Any system that relies on a partitioned database that can be rebalanced needs to know which 
node(s) to query when looking for a specific set of records. The problem of finding the right 
node(s) to query is a specific instance of a broader problem known as *service discovery*. 

There are a few high-level strategies to manage this problem:

- **Handle request resolution within the cluster**: Map incoming requests to any given node in 
  the cluster (e.g. via a round robin load balancer for distributed load), and let that node 
  return the correct data response. If the selected node happens to contain the partition(s) to 
  serve the request, then the data can be returned without any further communication - otherwise,
  the node is responsible for querying the cluster metadata to find the appropriate node(s), 
  forwarding the request to those nodes, collecting the response, and sending it back to the 
  request. 
- **Handle request resolution in a dedicated routing tier**: Handle incoming requests in a 
  dedicated routing tier that forwards the request to the appropriate node(s). The routing tier 
  does not query the data itself, and only handles request mapping.
- **Handle request resolution in the client**: Require the client to somehow identify the 
  correct node(s) to query, and let the client connect directly to any given node by surfacing 
  the appropriate IP addresses and ports (e.g. via DNS lookup).

In all of these approaches, the underlying problem still remains - a source of truth for mapping 
partitions to nodes needs to be kept up to date as the set of nodes changes and data is 
rebalanced. Examples of strategies for maintaining consensus on partition-node mappings include:

- **Dedicated external service**: Rely on a separate coordination service that subscribes to 
  each node in the cluster, listens for any updates to the node set and / or rebalancing, and 
  maintains a source of truth that can be queried (e.g. by cluster nodes, routing tier, or 
  clients). It is unclear how consistency would be maintained with this service and the 
  underlying database, so it's likely that databases may have backup strategies for request 
  mapping in case this separate service gets stale. 
- **Gossip protocol**: Rely on nodes communicating updates to one another when any updates to 
  partition mappings occur. This eliminates any dependency to an external service, but increases 
  complexity of the database - as with the above, consistency across nodes is still an issue, 
  and it's likely that a backup strategy is necessary here as well. 

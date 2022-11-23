# Distributed Data

## Reasons for distributing a database across machines

- **Scalability**: Spread read, write, and data loads across multiple machines if the load 
  parameter(s) grow beyond what one machine can handle
- **Availability**: Eliminate single points of failure by expanding your application to 
  operate across several machines, data centers, and geographies
- **Latency**: Serve users across the world more quickly by distributing your application across 
  geographies

## Strategies for scaling

### Shared-memory / vertical scaling

An application can respond to an increase in load by simply provisioning larger and larger 
machines. This approach is known as *vertical scaling* (alternatively, a *shared-memory 
architecture*). Under this approach, multiple pieces of hardware (CPU, RAM, disk) are stitched 
together by one operating system.

While this is the simplest solution, it is not very feasible for large-scale 
applications since:

- Bottlenecks across the shared-memory architecture mean that doubling the size / performance of 
  a machine does not guarantee twice the performance
- Costs grow faster than linearly, particularly as provisioned machines get larger
- Larger machines are clearly not distributed across data centers / geographies, thereby not 
  fully addressing the availability / latency goals outlined above

### Shared-disk

Another approach is to host the data across several machines with independent CPUs and 
RAM, but with a shared disk connected via a fast network. This approach is known as a 
*shared-disk architecture*.

A shared-disk architecture suffers from a number of issues relative to the shared-nothing 
approach described below:

- For data writes, the database must deal with contention and locking of records, since multiple 
  nodes can attempt to modify the same data (enforcing a partitioning scheme where a given 
  record is managed by only one machine effectively reduces to a more expensive implementation 
  of the shared-nothing approach).
- For data reads, the database is more likely to encounter cache misses, since each node is 
  responsible for serving the entire dataset (again, assuming no partitioning scheme).
- Shared disk resources can be exhausted, particularly with highly concurrent requests.

However, despite these drawbacks, this architecture may be the appropriate choice if a suitable 
partitioning key for the shared-nothing approach does not exist. For queries that span across 
partitions, the data co-location in a shared-disk architecture can improve performance. 

### Shared-nothing / horizontal scaling

A third option is to host the data across several machines that do not share any resources at 
all. This approach is known as *horizontal scaling* (alternatively, a *shared-nothing 
architecture*). Under this approach, data is typically *partitioned* across nodes such that a 
specific record is managed by exactly one node in the database cluster.

There are several benefits to this architecture:

- No special hardware is necessary, so you're free to choose any configuration that yields the 
  best performance to price ratio.
- Database writes for a record are handled by the same node and do not require any locking 
  mechanism.
- Database reads are faster, since the likelihood of a cache hit is higher and the data 
  to query is smaller relative to the shared-disk architecture (since the set of data is 
  restricted to a given partition, rather than the global dataset).

Note that the major performance benefits described above are predicated on being able to 
identify an appropriate partitioning / sharding key. A poor partition key will result in queries 
reading and writing data across nodes, which in turn will degrade performance.

## Strategies for distributing data across nodes

When dealing with multiple nodes in a shared-nothing architecture, there are two primary 
mechanisms to distribute data across nodes:

- **Replication**: Maintaining a copy of the same data across several nodes - useful for redundancy
- **Partitioning / sharding**: Splitting the data into disjoint sets that can be assigned to 
  different nodes 

These mechanisms are not mutually exclusive, and are commonly used in the same data system (i.e. 
a database can be partitioned and each partition can be replicated across multiple nodes).
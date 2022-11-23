# Chapter 2: Data Models and Query Languages

## Relational Model Versus Document Model

The relational model organizes data as *relations* (tables), where each relation is an unordered 
collection of *tuples* (rows). This model is useful for:

- Expressing many-to-one / many-to-many relationships between entities, since data can be 
  normalized (which eliminates data duplication / consistency issues)
- Building queries that join data across relations, as joins are a well-supported operation across 
  many RDBMSs (note that joins are not *impossible* in document databases, but the operation is 
  typically not optimized or immediately supported out of the box)
- Maintaining strong guarantees about your data schema  
  
The document model organizes data as *documents*, where each document can be viewed as an 
independent collection of arbitrarily structured data. The document is the atomic unit upon 
which any document database operates - reading any part of a document returns the whole 
document, and updating any part of the document requires a full document rewrite. This model is 
useful for:

- Expressing data that requires a more dynamic data model, without the rigidity of the 
  relational model
- Expressing and retrieving data with one-to-many relationships, as data can be easily nested

## Query Languages for Data

At a high level, there are two types of querying languages: imperative and declarative. 
Imperative languages require authors to define a series of operations to execute in a certain 
order, while declarative languages require authors to define the target outcome with no 
instruction on how that result is to be derived. 

Declarative querying has several advantages:

- **Readability**: Declarative querying tends to be more concise and readable
- **More friendly to performance optimizations**: Because no instructions were provided on how 
  to get a particular result, the query execution engine has more freedom on identifying and 
  applying optimizations and parallelization
- **Less likely to need refactors**: Updates to the execution engine or query optimizer do not 
  invalidate declarative queries, whereas imperative queries might need to be rewritten
- **Agnostic to environment**: Declarative queries do not need to make assumptions about the 
  environment they are in, and so can be applied to databases that are hosted on single or 
  multiple machines 

These advantages come at the expense of fewer guarantees. For example, databases supporting SQL 
typically do not make any guarantees on table row ordering - this is left as an implementation 
detail for the database to manipulate for query optimization.

Certain programming models like MapReduce are in between. MapReduce requires defining a map and 
reduce function, but the details of how data is shuffled across mappers and reducers is managed by 
the database implementation.

## Graph-Like Data Models

The relational model can handle simple cases of many-to-many relationships, but can get unwieldy 
as the number and types of relationships grow among your data (e.g. due to rigid table schemas 
and a variable number of joins to express a particular relationship). Graph data models that 
represent data as nodes and edges are more useful in these situations. 

Note that the node and edge types need not be homogenous - the same graph can have a number of 
different types of nodes (people, places, events, etc.) and edges (connections, check-ins, 
attendance, etc.). Facebook maintains a single graph with many such types for its data.

Two types of graph models worth knowing:

- **Property graph model**: Each node consists of a unique identifier, a set of outgoing edges, 
  a set of incoming edges, and a collection of key-value pairs (properties). Each edge consists 
  of a unique identifier, the tail vertex and head vertex (meaning the graph is directed), a 
  label to describe the relationship between the nodes (i.e. edge type), and a collection of 
  key-value pairs (properties). Since there is no restriction on the properties / labels a node 
  / edge can have or the types of nodes that can be connected, data and relationships can be 
  fully heterogenous.   
- **Triple store model**: Data is stored as three-part statements (subject, predicate, object), 
  such as (Jim, likes, bananas). The subject is equivalent to a node in a graph. The object is 
  either a primitive data type (in which case, the predicate and object are added as a key-value 
  property to the subject node) or another vertex (in which case, the predicate is a directed 
  edge in the graph). The end result is similar to the property graph model in that nodes are 
  connected by directed edges and can have arbitrary key-value properties. 
  
## SQL vs NoSQL for your system

### Performance of SQL (relational) vs NoSQL (document, graph) databases

Generally speaking, NoSQL databases are considered to be faster and more scalable, particularly 
under large data / write loads. This is due to the following:

- **Data locality**: Data in relational databases tends to be normalized, which means that 
  retrieving data often requires joins. While normalization eliminates data duplication and 
  consistency issues (e.g. in the case of making an update to a normalized field used across 
  many rows / tables), it also means that coupled data is not stored together. Data in 
  NoSQL databases is stored together, so reading a document is a relatively fast disk retrieval 
  (note that this benefit does not hold true if documents need to be frequently joined).
- **Relaxed ACID guarantees at write time**: ACID guarantees introduce overhead in write 
  operations (atomicity introduces locks across tables, consistency requires propagation of data 
  across nodes before completion, and durability requires a disk flush). NoSQL databases tend to 
  relax ACID requirements, which in turn favors performance. See the highest voted 
  [SE answer](https://softwareengineering.stackexchange.com/questions/194340/why-are-nosql-databases-more-scalable-than-sql) 
  for reference.
- **Conducive to horizontal scaling**: SQL databases are often used for their strong support of 
  ACID transactions, table joins, and referential integrity. However, these operations are more 
  complicated to support when table data is distributed across nodes (which would require 
  distributed table locks, synchronous replication across nodes, shard data moving across the 
  network, etc.). As stated above, NoSQL databases often forgo support for these operations in 
  favor of performance, which means that they are more conducive to horizontal scaling off the 
  shelf.
  
### Choosing SQL vs NoSQL for your system

The line between SQL and NoSQL database solutions is blurring. Consider that:

- Relational databases are starting to offer support for more open-ended data structures such as 
  JSON
- Google's Spanner, a relational database, allows you to explicitly declare tables to 
"interleave" amongst other tables in storage, to gain the data locality performance benefit 
without sacrificing normalization
- Several NoSQL databases offer varying levels of ACID guarantees now
- Horizontal scaling in SQL databases is possible (e.g. 
  [sharding](https://en.wikipedia.org/wiki/Shard_(database_architecture))) even without relaxing 
  functionality (e.g. [MySQL Cluster](https://www.mysql.com/products/cluster/scalability.html)), 
  and a SQL database configured to relax its ACID constraints, join support, and referential 
  integrity functionality should horizontally scale as well as NoSQL

With such developments, I think an appropriate decision framework would be to:

1. **Identify what operations your system needs to support**: What level of ACID guarantees are 
   necessary? Is join support necessary? How frequently do you expect data to change (the more 
   frequently it changes, the better it is to normalize that data - changes need only be applied to 
   one location without duplication / consistency issues)? How frequently do you expect schemas 
   to evolve?
2. **Identify what traffic your system needs to support**: How many reads, writes, and records 
   do you need to support in your database? If read-heavy, can you get away with caching or is 
   denormalizing data necessary? If write-heavy, can you rely on multiple leaders / in-memory 
   data stores, or is consistency / durability necessary? Is vertical scaling an 
   option, or is horizontal scaling necessary? These parameters can inform which ACID guarantees 
   are prohibitively expensive.
3. **Choose a sufficient database that minimizes impedance mismatch**: By sufficient, I mean to 
   choose a database that supports the operations and traffic identified in the first two steps. 
   By minimizing impedance mismatch, I mean to choose a database that most naturally conforms to 
   the actual data you're trying to store. This will lead to more natural query and application 
   logic.

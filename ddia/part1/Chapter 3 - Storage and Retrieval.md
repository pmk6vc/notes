# Chapter 3: Storage and Retrieval


## Data Structures That Power Your Database

There are two key data structures that are discussed in this section:

1. **Log**: Durable append-only set of files used to persist actual data (or references to 
   actual data somewhere else on disk).  
2. **Index**: Additional structure with metadata derived from the primary data, used to speed up 
   common data reads. Note that indexing involves a trade-off - maintaining indexes can speed up 
   reads but also incurs overhead on writes, as each index needs to be updated in response to 
   new data. 

The objective of this section will be to describe different index schemes , and identify 
the situations in which a particular scheme performs well and poorly. We'll start with examining
schemes for data that can be structured as *key-value pairs* (note that this includes 
document-like data as well as relational data with a primary key).

### Append-only segment files with in-memory hash index

The simplest starting point is to implement an append-only log as the storage engine and an 
in-memory hash map that maps each key to the location (e.g. as byte offset) in the log.

In order to avoid running out of disk space with an ever-increasing log, we can choose to break 
out the log as a series of immutable segment files, each of a certain size. We can then 
introduce a background process to periodically perform *compaction*, an operation that assembles 
a new segment file by pulling in only the latest valid keys from several old segments, and 
delete the old segments. To maintain an index for such a database, we can maintain an in-memory 
hash table for each segment file - reads must then iterate through each hash table for each 
segment file in reverse chronological order to find the data for a given key.

Considerations for implementing such a database include:

- **Serving reads and writes during compaction**: Reads can be directed to the old segment files 
  that are being compacted, and writes can be directed to the new segment file that is being 
  assembled
- **File format**: Might want to use a compact binary format to avoid using up more disk space
- **Deleting records**: Since the log / segments are append-only, we can introduce a special 
  type of record indicating deletion (i.e. a *tombstone* record). At compaction time, this record 
  can be interpreted as a deletion and be removed from the new segment file (unclear whether 
  this data is accessible before compaction - can remove the key to delete from the 
  segment-level hash map, perhaps?)
- **Crash recovery**: Database restarts will lose the segment-level hash maps. Rather than 
  rebuilding these by reading the actual segment files (which can be expensive and painful), we 
  can take periodic snapshots of the in-memory hash maps and dump them to disk, so they can be 
  loaded more quickly upon restart.
- **Partially written records**: If a database crashes while appending a record, the record may 
  only be partially written. To hedge against this risk, we can introduce checksums before and 
  after writes, to avoid data corruption.
- **Concurrency control**: Since writes need to be appended in a strictly sequential order, we 
  can simply choose to implement one writer thread. Reads can occur concurrently.

Strengths of this approach:

- Appending to a segment file and segment merging can take advantage of data locality on disk, 
  since data is written in an append-only fashion
- Concurrent reads and writes are easy to reason about, thanks to the immutability of segment 
  files and the sequential nature of append-only logs
- Crash recovery is simpler, thanks to the immutability of segment files

Drawbacks of this approach:

- Hash tables across all segments must fit in memory (disk-level hash maps are clunky and slow), so 
  this approach does not work if you have a very large keyspace.
- Range queries are not efficient, since there is no ordering on which key belongs to which 
  segment - we need to check every segment for every key in the requested range.
- Compaction might not be able to keep up with writes at high write throughputs.

Systems with this approach:

- Bitcask

### Sorted append-only segment files with in-memory hash indexes (SSTables & LSM-Trees)

The premise of the *Sorted String Table* is to sort the segment files in the previous method by 
key, as follows:

- Maintain an in-memory balanced tree data structure (e.g. red-black tree or AVL tree), known as a 
  **memtable**, that inserts incoming writes / updates with respect to the key. Use a 
  write-ahead log maintained on disk to rebuild the memtable in case the system crashes.
- When the memtable exceeds a certain size limit (typically a few megabytes), dump it to disk as 
  a segment file sorted by key. Incoming writes are written to a new memtable instance.
- To serve a read request, look up the key in the memtable and then subsequently in segment 
  files in reverse chronological order.
- As with the previous system, run a merging and compaction process in the background.

Strengths of this approach:

- Merging segments is simple and efficient, even if the segment size is greater than available 
  memory. A mergesort-style algorithm that consumes each line from each of the sorted segments 
  is sufficient to build a new sorted segment, without having segments completely in memory.
- The entire keyspace no longer needs to fit in memory. Segment-level hash tables can have an 
  evenly distributed sparse set of keys across the full segment - since keys are ordered, the 
  database can find an arbitrary key *k* simply by iterating through the disk block between the two 
  keys in the hash table that sandwich *k*. As long as there are enough keys in the hash table to 
  ensure that each block does not exceed a few kilobytes, lookups will be quick enough in practice.
- Since the database needs to iterate through the block between two keys anyways, these blocks 
  can be compressed to further minimize the footprint on the disk and network.
- Range queries are more efficient, since the memtable and each segment can be scanned for the 
  key range in parallel with a binary search-like lookup.

Drawbacks of this approach:

- Queries involving nonexistent keys require querying the memtable and every SSTable 
  before returning an empty set. This drawback can be mitigated with data structures like Bloom 
  filters.
- Compaction might not be able to keep up with writes at high write throughputs. 

Systems with this approach:

- [Google Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
- Cassandra
- HBase
- LevelDB
- RocksDB

### Sorted mutable pages with balanced tree index (B-Trees)

The premise of the B-tree indexing approach is as follows:

- Store data in mutable fixed-size *blocks* or *pages* of several kilobytes, sorted by key (as 
  opposed to SSTables, which store data in several megabyte immutable segments sorted by key).
- Each page is responsible for a certain continuous range of keys, and contains references to 
  other pages by their corresponding address on disk. The page references are balanced, meaning 
  each page has the same number of references to other pages (i.e. the same number of child 
  nodes). This number is known as the *branching factor*, and is typically several hundred.
- Each page becomes more granular in the range of keys it references as the depth of the tree 
  increases. A higher level node might reference a single page for all keys between, say, 100 
  and 200 - this page in turn will break up the 100-200 range into several more discrete page 
  references. Leaf pages provide the actual data associated with a specific key, either by 
  containing the value itself or by containing a final reference to pages where the value can be 
  found.
- Incoming writes will mutate the pages. For an existing key, the database will traverse the 
  B-tree to find the page where that key is defined, update the leaf page, and then write the 
  full page back to disk (the page is the atomic unit). For a new key, the database will 
  identify the leaf page where that new key belongs and add it to the page (if the page is full, 
  the database will splice the existing page into two and update the parent node with the new 
  addresses accordingly).
- Incoming writes are also written to an append-only write-ahead log to avoid issues with data 
  corruption during writes and page updates.

Strengths of this approach:

- Each key exists in exactly one place in the index, which makes B-trees attractive in databases 
  that want to offer strong transactional semantics
- With a high branching factor, even data measured in the terabytes need only have a balanced 
  tree of height 4 or 5, making traversal a cheap operation

Drawbacks of this approach:

- Inefficient disk usage as leaf pages might be fragmented
- Poor disk locality, since leaf pages may be written anywhere instead of sequentially

Systems with this approach:

- Most of them

### Other indexing structures

- **Secondary index**: An additional index on a non-primary key. This means that a key does 
  not uniquely specify a single record / value, which can be resolved either by appending a row 
  identifier to each key (to ensure uniqueness) or by returning the set of records with that 
  secondary key.
- **Clustered index**: An index that stores the actual record as the value for a given key, as 
  opposed to a reference. References are useful in the case of multiple indexes, to avoid data 
  duplication and inconsistency - in the case of references, the actual data is written to *heap 
  files*. They come at the cost of some performance, as an additional hop is needed from the 
  index to the appropriate heap file. Note that all the indexes above work whether the data is 
  written in the actual index (e.g. directly in pages / segment files) or in heap files, with 
  the B-tree / LSM index storing references to the heap.
- **Covering index**: Similar to a clustered index, except that only a subset of columns for a 
  given row are stored directly in the index - the remainder are written to heap files. For 
  frequently accessed columns, this may be a good compromise.
- **Multi-column index**: An index that concatenates several fields into one key for sorting 
  purposes.
- **In-memory databases**: Databases that avoid persisting data to disk in favor of the 
  performance benefits of storing data in memory. In-memory databases like Memcached and Redis are 
  sometimes used exclusively for caching, while others also aim for some level of durability.


## Transaction Processing or Analytics?

### OLTP vs OLAP

There are broadly two access patterns when it comes to interacting with a database:

- **Online Transaction Processing (OLTP)**: High-frequency low-latency reads and writes that 
  typically affect a small number of records. Characterizes user-facing systems. The bottleneck 
  tends to be disk seek time, which can be mitigated by techniques like indexing. Writes tend to 
  be driven by user input.
- **Online Analytic Processing (OLAP)**: Infrequent scans and aggregates for a subset of 
  columns over a large number of records. Characterizes retrospective business analysis. The 
  bottleneck tends to be disk bandwidth, which can be mitigated by moving over to columnar 
  storage formats. Writes tend to be driven by bulk ETL pipelines or event streams.

Organizations tend to set up separate systems for OLAP use cases (known as *data warehouses*), to 
avoid affecting performance of live, user-facing OLTP systems. This separation means that 
systems can do a better job catering to their specific use cases.

### Star and snowflake schemas

Data warehouses tend to be organized with a **star schema** (also known as **dimensional 
modeling**). In a star schema, the central entity is the *fact table*, where each row represents a 
discrete event or entity. Each row in the fact table is typically populated with foreign keys 
referencing various *dimension tables*.

A **snowflake schema** is an extension of this idea, where dimensions are further broken down 
into subdimensions, with even more normalization.

## OLAP Performance Improvement Strategies

### Column-Oriented Storage

Since queries on OLAP data warehouses are characterized by accessing a small subset of columns 
across a large number of rows, one idea for executing queries efficiently is to store the 
underlying data in a **column-oriented** fashion (as opposed to **row-oriented**, which is typical 
of OLTP systems). In a column-oriented system, column data is co-located within a file instead 
of row data. 

Benefits of column-oriented storage layout for the OLAP use case:

- Database does not need to seek through as many data files due to column data locality, saving 
  disk retrieval time
- Database does not need to load, parse, and discard column data that was never requested by the 
  query (which would be necessary if data was stored and loaded in a row-oriented manner), 
  saving disk retrieval and compute time
- Co-located column data lends itself well for efficient compression since data across rows 
  often has the same value for a given column / fact (e.g. date), saving disk space, network 
  bandwidth, and compute time (compressed data can be loaded directly into the CPU cache for 
  *vectorized processing*)

The column-oriented storage layout requires the preservation of row ordering across all column 
segment files so that rows can be correctly reassembled, which means that column files cannot be 
independently sorted for indexing. Rather, we can simply continue to build a row-level index 
(i.e. sort rows by a certain key) and then persist in a column-oriented storage format. With 
this approach, a read can be served by:

- Identifying the indices of the row(s) to return by querying the column file(s) defining the 
  sort key (a query that filters with dimensions not covered in the sort key will not be able to 
  take advantage of indexing regardless of row-oriented or column-oriented storage)
- Retrieving the data at those indices across each of the column files requested in the query

Writes to a compressed, sorted column-oriented storage system can be managed with the LSM-tree / 
SSTable paradigm described above. All writes first go to an in-memory balancing data structure 
based on the row sort key, and are merged across each column file once the in-memory 
tree is full. Compaction and column file cleanup can continue happening in the background, as 
with row-oriented storage. Note that a strategy relying on updates in place (e.g. B-trees) would 
not work for column-oriented storage, since inserting one row would require rewriting *all* the 
column files (instead of just one segment file in a row-oriented approach).

### Aggregation: Data Cubes and Materialized Views

Another lever for executing queries efficiently is to precompute aggregates across common 
dimensions, to serve as a cache for such queries. A *materialized view* is a precomputed 
query result that is persisted to disk (as opposed to a standard virtual view, which is just a 
shortcut for writing queries without any data persistence). A *data cube* is a special instance 
of a materialized view that stores aggregates across a user-defined set of dimensions - any 
aggregation queries relying on a subset of the cube's dimensions can be computed off of the cube 
instead.

Materialized views make writes more expensive, as aggregates need to be recomputed, but they may be 
effective for read-heavy data warehouses.  
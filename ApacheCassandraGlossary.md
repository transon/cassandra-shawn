## Terms ##

This is the glossary of terms that are important to understand when working with Apache Cassandra. There's a lot of really good material at http://wiki.apache.org/cassandra. But the first time you read the available Cassandra documentation or attempt to follow a thread on the user list can be tricky, as each new term seems to be explained only with other terms that are new too. Many of these concepts may be daunting to beginning or even intermediate web developers or database administrators, so it's presented here to offer an easy reference as you work.

This is just a glossary, so it's not intended to be read start to finish. Some information in this glossary is repeated and expanded upon in the relevant explanatory sections in the rest of the book.

  * Anti-Entropy

> Anti-entropy, or replica synchronization, is the mechanism in Cassandra for ensuring that data on different nodes are updated to the newest version.

> Here's how it works: during a major compaction (see Compaction), the server initiates a TreeRequest/TreeReponse conversation to exchange Merkle Trees with neighboring nodes. The Merkle Tree is a hash representing the data in that Column Family. If the trees from the different nodes don't match, then they have to be reconciled (or "repaired") in order to determine the latest data values they should all be set to. This tree comparison validation is the responsibility of the org.apache.cassandra.service.AntiEntropyService class. AntiEntropyService implements the Singleton pattern, and defines the static Differencer class as well, which is used to compare two trees and if it finds any differences it launches a repair for the ranges that don't agree.

> Anti-entropy is used in Amazon's Dynamo, and Cassandra's implementation is modeled on that (see Section 4.7 of the Dynamo paper).

> In Dynamo, they use a Merkle Tree for anti-entropy (see Merkle Tree). Cassandra does too, but the implementation is a little different. In Cassandra, each Column Family has its own Merkle Tree; the tree is created as a snapshot during a major compaction operation (see Compaction), and it is only kept as long as is required to send it to the neighboring nodes on the ring. The advantage of this implementation is that it reduces disk I/O.

> See Read Repair for more information on how these repairs occur.

  * Async Write

> Sometimes called "async writes" in documentation and user lists, this simply means "asynchronous writes", and refers to the fact that Cassandra makes heavy use of the java.util.concurrent library components such as ExecutorService and Future< T > for writing data to buffers.

  * Avro

> Avro is replacing Thrift as the RPC client for interacting with Cassandra. Avro is a sub-project of the Apache Hadoop project, created by Doug Cutting (creator of Hadoop and Lucene). It provides functionality similar to Thrift, but is a dynamic data serialization library that has an advantage over Thrift in that it does not require static code generation. Another reason that the project is migrating to Avro is that Thrift was originally created by Facebook and then donated to Apache, but since that time it has received little active development attention.

> This means that the Cassandra server will be ported from org.apache.cassandra.thrift.CassandraServer to org.apache.cassandra.avro.CassandraServer. As of this writing, this is underway and not yet complete.

> You can find out more about Avro at its project page, http://avro.apache.org.

  * Bigtable

> A key predecessor to Cassandra, Bigtable was created at Google in 2006 as a high-performance columnar database on top of Google File System (GFS).

> Cassandra inherits these aspects from Bigtable: **_sparse array data, and disk storage using an SSTable_**.

> Yahoo!'s HBase is a Bigtable clone.

> You can read the complete Google Bigtable paper at http://labs.google.com/papers/bigtable.html.

  * Bloom Filter

> In simple terms, a Bloom Filter is a very fast non-deterministic algorithm for testing whether an element is a member of a set. They are non-deterministic because it is possible to get a false-positive read from a Bloom Filter, but not a false-negative. Bloom Filters work by mapping the values in a data set into a bit array and condensing a larger dataset into a digest string. The digest, by definition, uses a much smaller amount of memory than the original data would.

> Cassandra uses Bloom Filters to reduce disk access, which can be expensive, on Key lookups. Every SSTable has an associated Bloom Filter; when a query is performed, the Bloom Filter is checked first before accessing disk. Because false-positives are not possible, if the filter indicates that the element does not exist in the set, it certainly doesn't; if they filter thinks that the element is in the set, then disk is accessed to make sure.

> While it is a disadvantage that false-positives are possible with Bloom Filters, their advantage is that they can be very fast because they use space efficiently, because (unlike simple arrays, hashtables, or linked lists) they do not store their elements completely. Instead, Bloom Filters make heavy use of memory, and reduce disk access. One result is that the number of false positive increases as the number of elements increases.

> Bloom Filters are used by Apache Hadoop, Google Bigtable and by Squid Proxy Cache. They are named for their inventor, Burton Bloom.

  * Cassandra

> In Greek mythology, Cassandra was the daughter of King Priam and Queen Hecuba of Troy. She was so beautiful that the god Apollo gave her the ability to see the future. But when she refused his amorous advances, he cursed her such that she would accurately predict everything that would happen, yet no one would believe her. Cassandra foresaw the destruction of her city of Troy, but was powerless to stop it. The Cassandra distributed database is named for her.

> The database itself is an Apache project available at http://cassandra.apache.org. It has the following key properties: it is decentralized, elastic, fault-tolerant, tune-ably consistent, highly available, and is designed to massively scale on commodity servers spread across different data centers. It is in use at companies such as Digg, Facebook, Twitter, Cloudkick, Cisco, IBM, Reddit, Rackspace, SimpleGeo, Ooyala, and OpenX.

> Cassandra was originally written at Facebook to solve their Inbox Search problem. The team was led by Jeff Hammerbacher, with Avinash Lakshman, Karthik Ranganathan, and Facebook engineer on the Search Team Prashant Malik as key engineers. The code was released as an open source Google Code project in July of 2008. In March of 2009 it was moved to an Apache Incubator project and on February 17 it was voted into a top-level project.

> A central paper on Cassandra by Facebook's Lakshman and Malik called "A Decentralized Structured Storage System" is available here: http://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf.

> A blog post from 2008 by Avinash Lakshman describes how they are using Cassandra at Facebook: http://www.facebook.com/note.php?note_id=24413138919&id=9445547199&index=9.

> It is perhaps easy to see why the Cassandra database is aptly named; its community asserts that Cassandra and other similar NoSQL databases are the future. Despite widespread use of eventually consistent databases at companies such as Amazon, Google, Facebook, and Twitter, there remain many skeptics ("non-believers") of such a model.

> The Java client Hector by Google's Ran Tavory is named for Cassandra's brother.

  * Chiton

> In ancient Greece, a chiton was a cloth garment, typically sleeveless, worn by both men and women. Chiton is the namesake for the open source project Chiton by Brandon Williams, which is a Python GTK-based browser for Apache Cassandra. It is currently hosted at http://github.com/driftx/chiton.

> A related project is Telephus, a low-level client API for Cassandra written in Twisted Python. It is currently hosted at http://github.com/driftx/Telephus.

  * Cluster

> A Cluster is two or more Cassandra instances acting in concert. These instances communicate with one another using Gossip.

> When you configure a new instance to introduce to your cluster, you'll need to do a few things. First, indicate a Seed Node. Next, you need to indicate the ports on which Cassandra should listen for two things: Gossip and the Thrift interface. Once your cluster is configured, use the Node Tool to verify that it is set up correctly.

  * Column

> A Column is the most basic unit of representation in the Cassandra data model. A column is a triplet of a name (sometimes referred to as a "key"), a value, and a timestamp. A Column's values, including the timestamp, are all supplied by the client. The data type for the name and value are Java byte arrays, typically represented as Strings. The data type for the timestamp is a long primitive. Columns are immutable in order to prevent multi-threading issues.

> Columns are organized into Column Families. Because Column Families are each stored in separate files, be sure to keep related Columns defined together in the same Column Family.

> The Column is defined in Cassandra by the org.apache.cassandra.db.IColumn interface, which allows a variety of operations, including getting the value of the column as a byte[.md](.md) or getting its Subcolumns as a Collection< IColumn >, and finding the time of the most recent change.

> Columns are sorted by their type, which is one of: AsciiType, BytesType, LexicalUUIDType, LongType, TimeUUIDType, UTF8Type.

> See also Column Family.

  * Column Family

> A Column Family is roughly analogous to a Table in a relational model. It is a container for an ordered collection of Columns.

> Because each Column Family is stored in a separate file, be sure to define Columns that you are likely to access together in the same Column Family.

> You define your application's Column Families in the Cassandra configuration file, storage-conf.xml. In this configuration, you can supply values for the size of the Row Cache, the size of the Key Cache, and the "Read Repair Chance". Column Families can be one of two types: Standard, or Super.

> The order of the columns within a row is configurable.

> See also Column, Keyspace, SuperColumn.

  * Column Name

> The name part of the name/value pair stored within a Row.

  * Column Value

> The value part of the name/value pair stored within a Row. The size of a column value is limited by machine memory.

  * Commit Log

> The commit log is responsible for all of the write operations in Cassandra. When you perform a write, it first enters the commit log so the data won't be lost in the event of failure; then the value is populated in the memtable so its available in memory for performance; once the Memtable fills up the data is flushed to the SSTable.

> It is represented by the org.apache.cassandra.db.commitlog.CommitLog class. On every write or delete, an entry in the form of a RowMutation object is serialized and appended to the commit log. These objects are organized into Commit Log Segments. Commit logs roll once they reach a size threshold of 128MB; when a new commit log is created, it accepts writes in transit.

> In order to guarantee durability for your Cassandra cluster, examine the settings in the server configuration file. By default, the CommitLogSync setting is periodic, which means that the server will make writes durable only every specific interval. When the server is set to periodically make writes durable, you can potentially lose the data that has not yet been synced to disk from the write-behind cache.

> You can change the value of the configuration attribute from periodic to batch, to specify that Cassandra must flush to disk before it acknowledges a write. Changing this value will require some taking performance metrics, as there is a necessary trade-off here; that is, forcing Cassandra to write more immediately constrains its freedom to manage its own resources. If you do set CommitLogSync to "batch", you need to provide a suitable value CommitLogSyncBatchWindowInMS, where MS is the number of milliseconds between each sync effort. Moreover, this is not generally needed in a multi-node cluster when using write replication, because replication by definition means that the write isn't acknowledged until another node has it.

> If you decide to use batch mode, you will probably want to split the commit log onto a separate device for performance reasons.

> If your commit log is set to "batch", then the commit log will block until the write is synced to disk.

  * Compaction

> Compaction is the process of freeing up space by merging large accumulated data files. This is roughly analogous to rebuilding a table in the relational world. On compaction the merged data is sorted, a new index is created over the sorted data, and the freshly merged, sorted, and indexed data is written to a single new file.

> The operations that are performed during compaction to free up space include: merging keys, combining columns, and deleting tombstones. This process is managed by the class org.apache.cassandra.db.CompactionManager. CompactionManager implements an MBean interface so it can be introspected.

> There are different types of compaction in Cassandra.

> A major compaction is triggered one of two ways: via a node probe, or automatically. A node probe sends a TreeRequest message to the nodes that neighbor the target. When a node receives a TreeRequest, it immediately performs a read-only compaction in order to validate the Column Family.

> A read-only compaction has the following steps:
```
       1. Get the key distribution from the column family.
       2. Once the rows have been added to the validator, if the Column Family needs to be validated, it will create the Merkle Tree and broadcast it to the neighboring nodes.
       3. The Merkle Trees are brought together in a "rendezvous" as a list of Differencers (trees that need validating or comparison).
       4. The comparison is executed by the StageManager class, which is responsible for handling concurrency issues in executing jobs. In this case, the StageManager uses an Anti-Entropy Stage. This uses the org.apache.cassandra.concurrent.JMXEnabledThreadPoolExecutor class which executes the compaction within a single thread, and makes the operation available as an MBean for inspection.
```

  * Compression

> Data compression on return is on the roadmap to be supported in future versions, but as of 0.6 it is not yet supported.

  * Consistency

> Consistency means that a transaction does not leave the database in an illegal state, that no integrity constrains are violated. This is considered a crucial aspect of transactions in relational databases and is one of the ACID properties (Atomic, Consistent, Isolated, Durable). In Cassandra, the relative degree of consistency can be calculated by the following:

> N = the number of nodes that store replicas of the data

> W = the number of replicas that must acknowledge receipt of a write before it can be said to be successful

> R = the number of replicas that are contacted when a data object is accessed in a read operation

> W + R > N = strong consistency

> W + R <= N = eventual consistency

  * Consistency Level

> This configurable setting allows you to decide how many replicas in the cluster must acknowledge a write operation or respond to a read operation in order to be considered successful. The Consistency Level is set according to your stated Replication Factor, and not the raw number of nodes in the cluster.

> There are multiple levels of consistency that you can tune for performance. The best performing level has the lowest consistency level. They mean different things for writing and reading.

> For write operations:
```
        * ZERO: Write operations will be handled in the background, asynchronously. This is the fastest way to write data, and the one that offers the least confidence that your operations will succeed.
        * ANY: This level was introduced in Cassandra 0.6, and means that you can be assured that your write operation was successful on at least one node, even if the acknowledgement is only for a hint (see Hinted Handoff). This is a relatively weak level of consistency.
        * ONE: Ensures that the write operation was written to at least one node, including its commit log and memtable. If a single node responds, the operation is considered successful.
        * QUORUM: A quorum is a number of nodes that represents consensus on an operation. It is determined by <ReplicationFactor> / 2 + 1. So if you have a replication factory of 10, then 6 replicas would have to acknowledge the operation to gain a quorum.
        * DCQUORUM: A version of Quorum that prefers replicas in the same Data Center in order to balance the high consistency level of Quorum with the lower latency of preferring to perform operations on replicas in the same Data Center.
        * ALL: Every node as specified in your <ReplicationFactor> configuration entry must successfully acknowledge the write operation. If any nodes do not acknowledge the write operation, the write fails. This has the highest level of consistency, and the lowest level of performance.
```

> For read operations:
```
        * ONE: This returns the value on the first node that responds. Performs a read repair in the background.
        * QUORUM: Queries all nodes and
        * ALL: Queries all nodes and returns the value with the most recent timestamp. This level waits for all nodes to respond, and if one doesn't, it fails the read operation.
```

> Note that there is no such thing as READ ZERO, as it doesn't make sense to specify that you at once want to read some data and don't need any node to respond.

  * Data Center Shard Strategy

> See Replication Strategy.

  * Decentralized

> Cassandra is considered decentralized because there it defines no master server, and instead uses a peer-to-peer approach in order to prevent bottlenecks and single points of failure. Decentralization is important in Cassandra, because it is what allows it to scale up and also to scale down; peers can enter or exit the cluster as they like, with minimal disruption. One way decentralization is realized in Cassandra is through Gossip.

  * Denormalization

> In relational databases, denormalization, or the creation of redundant data, is sometimes applied in order to improve performance of read-mostly applications such as in Online Analytical Processing (OLAP). In Cassandra, it is typical to see denormalized data, as this improves performance and helps account for the fact that data is structured according to the queries you'll need, in distinction to standard relational databases where the data is typically structured around the object model independently.
  * Durability

> When a database is durable, it means that writes will permanently survive, even in the event of a server crash or sudden power failure.

> Cassandra accomplishes durability by appending writes to the end of the commit log, allowing the server to avoid having to seek the location in the data file. Only the commit log needs to be synced with the file system, and this happens either periodically or in a specified batch window.

> When working in a single server node, Cassandra does not immediately synchronize a file's in-core state with storage device. That can mean if the server is shut down immediately after a write is performed, the write may not be present on restart. Note that a single server node is not recommended for production.

> See also (Commit Log).

  * Dynamo

> Created in 2006 by Amazon, and, with Google's Bigtable, a primary basis for Cassandra. From Dynamo, Cassandra gets the following: **_a key-value store, a symmetric peer-to-peer architecture, gossip-based discovery, eventual consistency, and tunable per operation_**.

> You can read the complete paper "Dynamo: Amazon's Highly-Available Key-Value Store" at http://www.allthingsdistributed.com/2007/10/amazons_dynamo.html.

  * Elastic

> Read and write throughput can increase linearly as more machines are added to the cluster.

  * Eventual Consistency

> Consistency is the property that describes the internal integrity of the data following an operation. In practical terms for a strongly consistent database, this means that once a client has performed a write operation, all readers will immediately see the new value. In eventual consistency, the database will not generally be consistent immediately, but rather eventually (where "eventually" is typically a matter of the small number of milliseconds it takes to send the new value to all replicas, relative to the amount of data, the number of nodes, and the geographical distribution of those nodes). DNS is an example of a popular eventually consistent architecture. Eventual consistency is sometimes called "weak consistency".

> Eventual consistency has become popular in the last few years because it offers the ability to support massive scalability. While it is possible to achieve high scalability in traditional fully consistent databases, the management overhead can become a burden. Of course, eventual consistency presents certain disadvantages, such as additional complexity in the programming model.

> Though the design of eventual consistency in Cassandra is based on how it is used in Amazon's Dynamo, Cassandra is probably better characterized as "tune-ably" consistent, rather than eventually consistent. That is, Cassandra allows you to configure the Consistency Level across the spectrum--including ensuring that Cassandra blocks until all replicas are readable.

> Riak, Voldemort, MongoDB, Yahoo!'s HBase, CouchDB, Microsoft's Dynomite, and Amazon's SimpleDB/Dynamo are other eventually consistent data stores.

  * Failure Detection

> Failure detection is the process of determining which nodes in a distributed fault-tolerant system have failed. Cassandra's implementation is based on the idea of Accrual Failure Detection, first advanced by the Japan Institute of Science and Technology in 2004. Accrual failure detection is based on two primary ideas: that failure detection should be flexible by being decoupled from the application being monitored, and that outputting a continuous level of "suspicion" regarding how confident the monitor is that a node has failed. This is desirable because it can take into account fluctuations in the network environment. Suspicion offers a more fluid and pro-active indication of the weaker or stronger possibility of failure based on interpretation (the sampling of heartbeats), as opposed to a simple binary assessment.

> Failure Detection is implemented in Cassandra by the org.apache.cassandra.gms.FailureDetector class.

> You can read the original Phi Accrual Failure Detection paper at http://ddg.jaist.ac.jp/pub/HDY+04.pdf.

  * Fault-Tolerant

> Fault tolerance is the aspect of a system to continue operating overall in the event of a failure of one or more of its components. Fault tolerance is also referred to as graceful degradation, meaning that if the system operation degrades following a failure, the degraded performance is relative only to the failed component(s).

  * Gossip

> The gossiper is responsible for ensuring that all of the nodes in a cluster are aware of the important state information in the other nodes. The gossiper ensures that even nodes that have failed or or not yet online are able to receive node states by running every second. It is designed to perform predictably even at sharply increased loads. The gossip protocol supports re-balancing of keys across the nodes, and supports Failure Detection. Gossip is an important part of the Anti-Entropy strategy.

> The state information that the gossiper shares is structured as key/value pairs. In Cassandra, the gossip protocol continues to gossip state information to other nodes until it is made obsolete by newer data.

> When a server node is started, it registers itself with the gossiper. For more information, check out the org.apache.cassandra.service.StorageService class.

> Also see the Amazon paper on Gossip at http://www.cs.cornell.edu/home/rvr/papers/flowgossip.pdf.

  * Hector

> An open source project created by Ran Tavory of Google and hosted at Git Hub, Hector is a Cassandra client written in Java. It wraps Thrift and offers JMX, connection pooling, and fail over.

  * Hinted Handoff

> This is a mechanism to ensure availability, fault-tolerance and graceful degradation. If a write operation occurs, and a node that is intended to receive that write goes down, a note (the "hint") is given ("handed off") to a different live node to indicate that it should replay the write operation to the unavailable node when it comes back online. This does two things: it reduces the amount of time that it takes for a node to get all of the data it missed once it comes back online; it also improves write performance in lower consistency levels. That is, a hinted handoff does not count as a sufficient acknowledgement for a write operation if the consistency level is set to ONE, QUORUM, or ALL. A hint does count as a write for Consistency Level ANY, however. Another way of putting this is that hinted writes are not readable in and of themselves.

> The node that received the hint will know very quickly when the unavailable node comes back online again, because of Gossip. If, for some reason, the hinted handoff doesn't work, the system can still perform a read repair.

```
1) A note is given to a different live node, it should replay the write opertion to the unavailable node when it comes back online, because of Gossip.
2) A hinted handoff acknowledgement does count as a write for CL = ANY.
3) Improve write performance. 
```

  * Key

> See Row Key.

  * Keyspace

> A Keyspace is a container for Column Families. It is roughly analogous to the database in the relational model, used in Cassandra to separate applications. Where a relational database is a collection of tables, a Keyspace is an ordered collection of Column Families. You define your application's Keyspace in the Cassandra configuration file, storage-conf.xml. When you define a Keyspace, you can also define its replication factor and its replica placement strategy. Within a given Cassandra cluster, you can have one or more Keyspaces, one for each application that you define.

> See also (Column Family).

  * Lexicographic Ordering

> Lexicographic ordering is the natural (alphabetic) ordering of the product of two ordered Cartesian sets.

  * Memtable

> An in-memory representation of data that have been recently written . Once the memtable is full, it is flushed to disk as an SSTable.

  * Merkle Tree

> Perhaps better known as a "Hash Tree", a Merkle Tree is a binary tree data structure that summarizes in short form the data in a larger data set. In a Hash Tree, the leaves are the data blocks (typically files on a file system) to be summarized. Every parent node in the tree is a hash of its direct child node, which tightly compacts the summary.

> In Cassandra, the Merkle Tree is implemented in the org.apache.cassandra.utils.MerkleTree class.

> Merkle Trees are used in Cassandra to ensure that the peer to peer network of nodes receives data blocks unaltered and unharmed. They are used in cryptography as well to verify the contents of files and transmissions, and are used in the Google Wave product. They are named for their inventor, Ralph Merkle.

  * Multiget

> Query by column name for a set of keys.

  * Multiget Slice

> Query to get a subset of columns for a set of keys.

  * Node

> An instance of Cassandra. Typically a Cassandra cluster will have many nodes, which is sometimes called a node ring.

  * Node Tool

> This is an executable file with the path bin/nodetool that inspects a cluster to determine whether it is properly configured, and to perform a variety of maintenance operations. The commands available on the nodetool are: cleanup, clearsnapshot, compact, cfstats, decommission, drain, flush, info, loadbalance, move, repair, ring, snapshot [snapshotname](snapshotname.md), removetoken, tpstats.

> For example, you may use nodetool drain to prevent the commit log from accepting any new writes.

  * NoSQL

> "NoSQL" is a general name for the collection of databases that do not use SQL (Structured Query Language) or a relational data model. It is used sometimes to mean "Not Only SQL" to indicate that the proponents of various non-relational databases do not suggest that relational databases are a bad choice--but rather that they are not the only choice for data storage.

  * Order-Preserving Partitioner

> This is a kind of Partitioner that has the following qualities: rows are stored by key order, aligning the physical structure of the data with your sort order. Configuring your Column Family to use Order-Preserving partitioning allows you to perform Range Slices, meaning that Cassandra knows what nodes have which keys.

> This partitioner is somewhat the opposite of the Random Partitioner: it has the advantage of allowing for efficient range queries, but the disadvantage of unevenly distributing keys.

> The Order-Preserving Partitioner (OPP) is implemented by the org.apache.cassandra.dht.OrderPreservingPartitionerclass.

> There is a special kind of OPP, called Collating Order-Preserving Partitioner (COPP). This acts like a regular OPP, but sorts the data in a collated manner according to English/US lexicography instead of byte ordering. For this reason, it is useful for locale-aware applications. The COPP is implemented by the org.apache.cassandra.dht.CollatingOrderPreservingPartitioner class.

> This is implemented in Cassandra by org.apache.cassandra.dht.OrderPreservingPartitioner.

> See Token.

  * Partition

> In general terms, a partition refers to a network partition, which is a break in the network that prevents one machine from interacting directly with another. A partition can be caused by failed switches, routers, or network interfaces. Consider a cluster of five machines {A, B, C, D, E} where {A, B} are on one subnet and {C, D, E} are on a second subnet. If the switch to which {C, D, E} are connected fails, then you have a network partition that isolates the two sub-clusters {A, B} and {C, D, E}.

> Cassandra is a fault-tolerant database, and network partitions are one such fault. As such, it is able to continue operating in the face of a network partition, and merge data in replication once the partition is healed again.

  * Partitioner

> The Partitioner controls how your data are distributed over your nodes. In order to find a set of keys, Cassandra must know what nodes have the range of values you're looking for. There are two types of partitioner: Order-Preserving Partitioner and Random Partitioner. You configure this in storage-conf.xml, using the < Partitioner > element: < Partitioner >org.apache.cassandra.dht.RandomPartitioner< /Partitioner >. Note that partitioning applies to the sorting of row keys, not columns.

> Once you have chosen a partitioner type, you cannot change it without destroying your data (because an SSTable is immutable).

> See Order-Preserving Partitioner and Random Partitioner.

  * Quorum

> A majority of nodes that respond to an operation. This is a configurable consistency level. In a quorum read, the proxy waits for a majority of nodes to respond with the same value; while this makes for a slower read operation, it also helps ensure that you don't get returned stale data.

  * Rack Aware Strategy

> See Replication Strategy.

  * Random Partitioner

> This is a kind of Partitioner that uses a BigIntegerToken with an MD5 hash to determine where to place the keys on the node ring. This has the advantage of spreading your keys evenly across your cluster, but the disadvantage of causing inefficient range queries. This is the default partitioner.

> See Partitioner and Random Partitioner.

  * Range Slice

> Query to get a subset of columns for a range of keys.

  * Read Repair

> This is another mechanism to ensure consistency throughout the node ring. In a read operation, if Cassandra detects that some nodes have responded with data that is inconsistent with the response of other newer nodes, it makes a note to perform a read repair on the old nodes. The read repair means that Cassandra will send a write request to the nodes with stale data to get them up to date with the newer data returned from the original read operation. It does this by pulling all of the data from the node and performing a merge, and then writing the merged data back to the nodes that were out of sync. The detection of inconsistent data is made by comparing timestamps and checksums.

> The method for reconciliation is the org.apache.cassandra.streaming package.

  * Replication

> In general distributed systems terms, replication refers to storing multiple copies of data on multiple machines so that if one machine fails or becomes unavailable due to a Partition, then the cluster can still make data available. To help understand this, in general terms, caching is a simple form of replication. In Cassandra, replication is a means of providing high performance and availability/fault-tolerance.

  * Replication Factor

> Cassandra offers a configurable replication factor, which allows you essentially to decide how much you want to pay in performance to gain more consistency. That is, your consistency Level for reading and writing data is based on the Replication Factor, as it refers to the number of nodes across which you have replicated data. Replication Factor is configured using the < ReplicationFactor > element in storage-conf.xml.

> See Consistency Level.

  * Replication Strategy

> The replication strategy, sometimes referred to as the placement strategy, determines how replicas will be distributed. The first replica is always placed in the node claiming the key range of its Token. All remaining replicas are distributed according to a configurable replication strategy.

> The Gang of Four Strategy pattern is employed to allow pluggable means of replication, but Cassandra comes with three out of the box. Choosing the right replication strategy is important, because in determining which nodes are responsible for which key ranges, you're also determining which nodes should receive write operations; this has a big impact on efficiency in different scenarios. The variety of pluggable strategies allows you greater flexibility, so that you can tune Cassandra according to your network topology and needs.

> The simplest replication strategy is called RackUnawareStrategy. As the name states, it is ignores rack placement (that is, it places replicas on the nearest nodes on the ring whether or not they are in the same rack in the data center).

> A more sophisticated and popular replication strategy is RackAwareStrategy. This strategy is important when you have more than one data center housing nodes in your Cassandra cluster, as it does two things: remember that the first node is automatically placed based on the token; this strategy will then place the second node in a different data center than the first node. It will then place the remaining nodes across different racks in the same data center.

> A third replication strategy is DatacenterShardStrategy. This allows you to specify more evenly than the RackAwareStrategy how replicas should be placed across data centers. To use it, you supply a file called datacenters.properties in which you indicate the desired replication strategy for each data center. This file is read and executed by the org.apache.cassandra.locator.DataCenterShardStrategy class.

> Replication Strategies are an extension of the org.apache.cassandra.locator.AbstractReplicationStrategy class. You can write your own replication strategy if you like by extending that class.

  * Row

> In a Column Family, a Row is a sorted map that matches column names to column values. In a Super Column, a Row is a sorted map that matches Super Column names to maps matching Column names to Column values. The Row Key defines the individual row, and the row defines the name/value pairs of the columns. The size of a single Row cannot exceed the amount of space on disk.

> Rows are sorted by their Partitioner, which is one of these types: RandomPartitioner, OrderPreservingPartitioner, or CollatingOrderPreservingPartitioner.

> Rows are defined by the class org.apache.cassandra.db.Row.

> See Row Key.

  * Row Key

> Sometimes called simply "Key", a Row Key is analogous to a primary key for an object in the relational model. It represents a way to identify a single Row of Columns, and is an arbitrary length string.

> In the Thrift interface, the Java client always assumes that Row Keys are encoded as UTF-8, but this is not the case for clients in other languages, where you may need to manually encode ASCII strings as UTF-8.

  * SEDA (Staged Event Driven Architecture)

> Cassandra employs a Staged Event Driven Architecture to gain massive throughput under highly concurrent conditions. SEDA attempts to overcome the overhead associated with threads. This overhead is due to scheduling, lock contention, and cache misses. The effect of SEDA is that work is not started and completed by the same thread, which can make a more complex code base, but also yield better performance. Therefore, much of the key work in Cassandra, such as Reading, Mutation, Gossiping, memtable flushing, and Compaction, are performed as stages (the "S" in SEDA). A stage is essentially a separated event queue.

> As events enter the incoming queue, the event handler supplied by the application is invoked. The controller is capable of dynamically tuning the number of threads allocated to each stage as demand dictates.

> The advantages of SEDA are higher concurrency and better management of CPU, disk, and network resources.

> You can read more about SEDA as it was originally proposed by Matt Welsh, David Culler, and Eric Brewer at http://www.eecs.harvard.edu/~mdw/proj/seda.

> Also see Stage (Stage).

  * Seed Node

> A seed is a node that already exists in a Cassandra cluster, which is used by newly added nodes to get up and running. A seed is another node for the new one to start gossiping with to get state information and learn the topology of the node ring; there may be many seeds in a cluster.

> A seed is configured in storage-conf.xml like this:
```
    <Seeds><Seed>10.0.1.1</Seed></Seeds>
```

  * Slice

> This is a type of read query. Use get\_slice() to query by a single column name or a range of column names. Use get\_range\_slice() to return a subset of columns for a range of keys.

  * Snitch

> A snitch is Cassandra's way of mapping a node to a physical location in the network. It helps determine the location of a node relative to another node in order to assist with discovery and ensure efficient request routing. There are different kinds of snitches.

> The EndpointSnitch (or RackInferringSnitch)For instance, it determines whether two nodes are in the same data center, or in the same rack. Its strategy for doing so is essentially to guess at the relative distance of two nodes in a data center and rack based on reading the second and third octets of their IP addresses.

> The DataCenterEndpointSnitch allows you to specify IP subnets for racks, grouped by which data center the racks are in.

> The PropertyFileSnitch allows you to map IP addresses to rack and data centers in a properties file called cassandra-rack.properties.

> The snitch strategy classes can be found in the org.apache.cassandra.locator package.

  * Sparse

> In the relational model, every data type (table) must have a value for every column, even if that value is sometimes null. Cassandra, on the other hand, represents a sparse or "schema-free" data model, which means that rows may have values for as many or as few of the defined Columns as you like. This allows for a degree of efficiency. For example, consider a 1000 x 1000 cell spreadsheet, similar to a relational table: if many of the cells have empty values, the storage model is inefficient.

  * SSTable

> SSTable stands for "Sorted String Table". Inherited from Google's Bigtable, an SSTable is how data is stored on disk in Cassandra. It is a log that only allows appending. In-memory tables (memtables) are used in front of SSTables for buffering and sorting of data. SSTables allow for high performance on writes, and can be compacted (see Compaction).

> SSTables are immutable. Once a memtable is flushed to disk as an SSTable, it cannot be changed by the application; Compaction only changes their on-disk representation.

> To import or export data from JSON (JavaScript Object Notation), check out the classes org.apache.cassandra.tools.SSTableImporter and SSTableExporter.

  * Stage

> Part of Cassandra's Staged Event Driven Architecture (SEDA), a stage wraps a basic unit of work. Stages are an important part of how Cassandra can perform well. A single operation may flow between various stages to complete, rather than getting completed in the same thread that started the work.

> A stage consists of an incoming event queue, an event handler, and an associated thread pool. Stages are managed by a controller that determines scheduling and thread allocation; Cassandra implements this kind of concurrency model using the thread pool java.util.concurrent.ExecutorService. To see specifically how this works, check out the org.apache.cassandra.concurrent.StageManager class.

> The following operations are represented as stages in Cassandra:
```
        * Read
        * Mutation
        * Gossip
        * Response
        * Anti-Entropy
        * Load Balancer
        * Migration
        * Streaming
```
> There are a few additional operations are implemented as stages too, including working with memtables in the ColumnFamilyStore class, and the Consistency Manager is a stage in the StorageService.

> An operation may start with one thread, which then hands off the work to another thread, and may hand it off to other threads. This handing-off is not directly between threads, however; it occurs between stages.

> Also see SEDA (Staged Event Driven Architecture) (SEDA (Staged Event Driven Architecture)).

  * Strong Consistency

> For reads, strong consistency means that If it is detected that a repair needs to be made, first perform the read repair, then return the result.

  * SuperColumn

> A SuperColumn is a Column whose value is not a string, but instead a map of other Columns, which in this context are called Subcolumns. The Subcolumns are ordered and the number of Columns you can define is unbounded. SuperColumns also differ from regular Columns in that they do not have an associated timestamp.

> SuperColumns are not recursive; that is, they only go one level deep; a SuperColumn can only hold a map of other Columns, and not a map of more SuperColumns.

> They are defined in SuperColumn.java, which implements both the IColumn and IColumnContainer interfaces. The interface allows you to perform a variety of operations, including the following: get all of the Subcolumns in a SuperColumn, get a single Subcolumn by name, add a Subcolumn, remove a Subcolumn, check the number of Subcolumns in this SuperColumn, and check when a Subcolumn was last modified.

> SuperColumns were one of the updates added by Facebook to the original data model of Google's Bigtable.

> See also Column Family.

  * Thrift

> Thrift is the name of the RPC client used to communicate with the Cassandra server. It statically generates an interface for serialization in a variety of languages, including C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, Smalltalk, and OCaml. It is this mechanism that allows you to interact with Cassandra from any of these client languages.

> It was created in April 2007 at Facebook and donated to Apache as an incubator project in May 2008. At the time of this writing, the Thrift interface is being replaced by the newer and more active Apache project Avro. Another advantage of Avro is that it does not require static code generation.

> You can read more about Thrift on its project page at http://incubator.apache.org/thrift.

  * Timestamp

> In Cassandra, timestamps for Column values are supplied by the client, so it is important to synchronize client clocks. The timestamp is by convention the number of microseconds since the Unix epoch (midnight, January 1, 1970).

  * Token

> Each node in the node ring has a single token which is used to claim a range of keys, based on the value of the token in the previous node in the ring. The representation of the token is dependent on the kind of partitioner used.

> With a Random Partitioner, then the token is an integer in the range 0-2127, generated by applying an MD5 hash on keys. This is represented by the org.apache.cassandra.dht.BigIntegerToken class.

> With an Order-Preserving Partitioner, the token is a UTF-8 string, based on the Key. This is represented by the org.apache.cassandra.dht.StringToken class.

> Tokens are represented in Cassandra by the org.apache.cassandra.dht.Token class.

  * Tombstone

> Cassandra does not immediately delete data following a delete operation, for performance reasons. Instead, it marks the data with a "tombstone", an indicator that the column has been deleted but not removed entirely yet. The tombstone can then be propagated to other replicas.

> Tombstones are discarded on major Compaction.

  * Vector Clock

> Vector clocks allow for partial, causal ordering of events in a distributed system. A vector clock maintains an array of logical clocks, one for each process, and each process contains a local copy of the clock.

> In order to keep the entire set of processes in a consistent logical state, one process will send its clock to another process which is then updated. In order to ensure consistency, some version of the following steps are typically followed:

> All clocks start at 0. Each time a process experiences an event, its clock is incremented by 1. Each time a process prepares to send a message, this too counts as an event, so it increments its clock by 1, and then sends its entire vector to the external process along with the message. Each time a process receives a message, this too counts as an event, so it therefore updates its own clock by 1, and then compares its vector to the vector wrapped in the incoming message from the external process. It updates its own clock with the maximum value from the comparison.

> A Vector Clock event synchronization strategy will likely be introduced in Cassandra 0.7.

  * Weak Consistency

> For reads, weak consistency improves performance by first returning results, and afterwards performing any necessary Read Repair.
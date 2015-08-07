虽然Cassandra: The Definitive Guide还没有出版，很期待啊！但是从其目录中可以了解Cassandra的切入点，先对每一个feature理论上有一个好的理解，然后再映射到源代码中理解其实现方式。

Table of Contents

---


# Copyright #

# Dedication #

# Foreword #

# Preface #

## Chapter 1. Introducing Cassandra ##


  * Section 1.1. Overview


  * Section 1.2. What's Wrong with Relational Databases?


  * Section 1.3. A Quick Review of Relational Databases


  * Section 1.4. The Cassandra Elevator Pitch


  * Section 1.5. Where Did Cassandra Come From?


  * Section 1.6. Use Cases for Cassandra


  * Section 1.7. Who Is Using Cassandra?


  * Section 1.8. Summary

## Chapter 2. Installing Cassandra ##


  * Section 2.1. Overview


  * Section 2.2. Installing the Binary


  * Section 2.3. Building from Source


  * Section 2.4. Running Cassandra


  * Section 2.5. Running the Command Line Client Interface


  * Section 2.6. Basic CLI Commands


  * Section 2.7. Summary

## Chapter 3. The Cassandra Data Model ##


  * Section 3.1. Overview


  * Section 3.2. The Relational Data Model


  * Section 3.3. A Simple Introduction


  * Section 3.4. Clusters


  * Section 3.5. Keyspaces


  * Section 3.6. Column Families


  * Section 3.7. Columns


  * Section 3.8. SuperColumns


  * Section 3.9. Design Differences Between RDBMS and Cassandra


  * Section 3.10. Design Patterns


  * Section 3.11. Some Things to Keep in Mind


  * Section 3.12. Summary

## Chapter 4. Sample Application ##


  * Section 4.1. Overview


  * Section 4.2. Data Design


  * Section 4.3. Hotel App RDBMS Design


  * Section 4.4. Hotel App Cassandra Design


  * Section 4.5. Hotel Application Code


  * Section 4.6. Twissandra


  * Section 4.7. Summary

## Chapter 5. The Cassandra Architecture ##


  * Section 5.1. Overview


  * Section 5.2. System Keyspace


  * Section 5.3. Peer-to-Peer


  * Section 5.4. Gossip & Failure Detection


  * Section 5.5. Anti-Entropy & Read Repair


  * Section 5.6. Memtables, SSTables, and Commit Logs


  * Section 5.7. Hinted Handoff


  * Section 5.8. Compaction


  * Section 5.9. Bloom Filters


  * Section 5.10. Tombstones


  * Section 5.11. Staged Event-Driven Architecture (SEDA)


  * Section 5.12. Managers and Services


  * Section 5.13. Summary

## Chapter 6. Configuring Cassandra ##


  * Section 6.1. Overview


  * Section 6.2. Keyspaces


  * Section 6.3. Replicas


  * Section 6.4. Replica Placement Strategies


  * Section 6.5. Replication Factor


  * Section 6.6. Partitioners


  * Section 6.7. Snitches


  * Section 6.8. Creating a Cluster


  * Section 6.9. Dynamic Ring Participation


  * Section 6.10. Security


  * Section 6.11. Miscellaneous Settings


  * Section 6.12. Additional Tools


  * Section 6.13. Summary

## Chapter 7. Reading and Writing Data ##


  * Section 7.1. Overview


  * Section 7.2. Query Differences Between RDBMS and Cassandra


  * Section 7.3. Basic Write Properties


  * Section 7.4. Consistency Levels


  * Section 7.5. Basic Read Properties


  * Section 7.6. The API


  * Section 7.7. Setup and Inserting Data


  * Section 7.8. Using a Simple Get


  * Section 7.9. Seeding Some Values


  * Section 7.10. Slice Predicate


  * Section 7.11. Get Range Slices


  * Section 7.12. Multiget Slice


  * Section 7.13. Deleting


  * Section 7.14. Batch Mutates


  * Section 7.15. Programmatically Defining Keyspaces and Column Families


  * Section 7.16. Summary

## Chapter 8. Clients ##


  * Section 8.1. Overview


  * Section 8.2. Basic Client API


  * Section 8.3. Thrift


  * Section 8.4. Avro


  * Section 8.5. A Bit of Git


  * Section 8.6. Connecting Client Nodes


  * Section 8.7. Web Console (Spring)


  * Section 8.8. Hector (Java)


  * Section 8.9. HectorSharp (C#)


  * Section 8.10. Chiton (Python)


  * Section 8.11. Pelops (Java)


  * Section 8.12. Kundera


  * Section 8.13. Fauna (Ruby)


  * Section 8.14. Summary

## Chapter 9. Monitoring ##


  * Section 9.1. Overview


  * Section 9.2. Logging


  * Section 9.3. Overview of JMX and MBeans


  * Section 9.4. Interacting with Cassandra via JMX


  * Section 9.5. Cassandra's MBeans


  * Section 9.6. Custom Cassandra MBeans


  * Section 9.7. Runtime Analysis Tools


  * Section 9.8. Health Check


  * Section 9.9. Summary

## Chapter 10. Maintenance ##


  * Section 10.1. Overview


  * Section 10.2. Getting Ring Information


  * Section 10.3. Getting Statistics


  * Section 10.4. Basic Maintenance


  * Section 10.5. Snapshots


  * Section 10.6. Load Balancing the Cluster


  * Section 10.7. Decommissioning a Node


  * Section 10.8. Updating Nodes


  * Section 10.9. Using Cluster Tool

## Chapter 11. Performance Tuning ##


  * Section 11.1. Overview


  * Section 11.2. Data Storage


  * Section 11.3. Reply Timeout


  * Section 11.4. Commit Logs


  * Section 11.5. Memtables


  * Section 11.6. Concurrency


  * Section 11.7. Caching


  * Section 11.8. Buffer Sizes


  * Section 11.9. Using the Python Stress Test


  * Section 11.10. Startup and JVM Settings


  * Section 11.11. Summary

## Chapter 12. Integrating Hadoop ##


  * Section 12.1. Overview


  * Section 12.2. What is Hadoop?


  * Section 12.3. Working with MapReduce


  * Section 12.4. Running the Word Count Example


  * Section 12.5. Tools Above MapReduce


  * Section 12.6. Cluster Configuration


  * Section 12.7. Use Cases


  * Section 12.8. Summary

## Appendix A. The NoSQL Landscape ##


  * Section A.1. Introduction


  * Section A.2. Non-Relational Databases


  * Section A.3. Object Databases


  * Section A.4. XML Databases


  * Section A.5. Document-Oriented Databases


  * Section A.6. Graph Databases


  * Section A.7. Key-Value Stores / Distributed Hashtables


  * Section A.8. Columnar Databases


  * Section A.9. NoSQL Summary


  * Section A.10. Summary

## Appendix B. Cassandra Glossary ##


  * Section B.1. Glossary


Colophon
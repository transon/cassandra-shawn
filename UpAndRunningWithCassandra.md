Cassandra is a hybrid non-relational database in the same class as Google's BigTable. It is more featureful than a key/value store like Riak, but supports fewer query types than a document store like MongoDB.

Cassandra was started by Facebook and later transferred to the open-source community. It is an ideal runtime database for web-scale domains like social networks.

This post is both a tutorial and a "getting started" overview. You will learn about Cassandra's features, data model, API, and operational requirements—everything you need to know to deploy a Cassandra-backed service.

May 11, 2010: post updated for Cassandra gem 0.8 and Cassandra version 0.6.
features

There are a number of reasons to choose Cassandra for your website. Compared to other databases, three big features stand out:

  * Flexible schema: with Cassandra, like a document store, you don't have to decide what fields you need in your records ahead of time. You can add and remove arbitrary fields on the fly. This is an incredible productivity boost, especially in large deployments.
  * True scalability: Cassandra scales horizontally in the purest sense. To add more capacity to a cluster, turn on another machine. You don't have restart any processes, change your application queries, or manually relocate any data.
  * Multi-datacenter awareness: you can adjust your node layout to ensure that if one datacenter burns in a fire, an alternative datacenter will have at least one full copy of every record.

Some other features that help put Cassandra above the competition :

  * Range queries: unlike most key/value stores, you can query for ordered ranges of keys.
  * List datastructures: super columns add a 5th dimension to the hybrid model, turning columns into lists. This is very handy for things like per-user indexes.
  * Distributed writes: you can read and write any data to anywhere in the cluster at any time. There is never any single point of failure.

Ref: http://blog.evanweaver.com/articles/2009/07/06/up-and-running-with-cassandra/
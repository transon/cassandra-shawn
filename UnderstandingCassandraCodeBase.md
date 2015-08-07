## Reading ##

Next you want to read some of the background material, depending on what part exactly you want to work on. I wanted to understand the read path, write path and how values are deleted, so I read the following documents about 5 times each. Yes, 5 times. Each. They are packed with information and I found myself absorbing a few more details each time I read. I used to read the document, get back to the source code, make sure I understand how the algorithm maps to the methods and classes, reread the document, reread the source code, read the unit tests (and run them, with a debugger) etc. Here are the docs.

<Architecture Internals>: http://wiki.apache.org/cassandra/ArchitectureInternals

<SEDA paper>: http://www.eecs.harvard.edu/~mdw/papers/seda-sosp01.pdf

<Hinted Handoff>: http://wiki.apache.org/cassandra/HintedHandoff

<Architecture AntiEntropy>: http://wiki.apache.org/cassandra/ArchitectureAntiEntropy

<Architecture SSTable>: http://wiki.apache.org/cassandra/ArchitectureSSTable

<Architecture CommitLog>: http://wiki.apache.org/cassandra/ArchitectureCommitLog

<Distributed Deletes>: http://wiki.apache.org/cassandra/DistributedDeletes

I also read the google BigTable paper http://labs.google.com/papers/bigtable.html and the fascinating Amazon’s Dynamo paper http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf, but that was a long time ago. They are good as background material, but not required to understand actual bits of code.

Well, after having read all this I was starting to get a clue what can be done and how but I still didn’t feel I’m at the level of really coding new features. After reading through the code a few times I realized I’m kind of stuck and still don’t understand things like “how do values really get deleted”, which class is responsible for which functionality, what stages are there and how is data flowing between stages, or “how can I mark and entire column family as deleted”, which is what I really wanted to do with the truncate operation.
Stages

Cassandra operates in a concurrency model described by the SEDA paper. This basically means that, unlike many other concurrent systems, an operation, say a write operation, does not start and end by the same thread. Instead, an operation starts at one thread, which then passes it to another thread (asynchronously), which then passes it to another thread etc, until it ends. As a matter of fact, the operation doesn’t exactly flow b/w threads, it actually flows b/w stages. It moves from one stage to another. Each stage is associated with a thread pool and this thread pool executes the operation when it’s convenient to it. Some operations are CPU bound, some are disk or network bound, so “convenience” is determined by resource availability. The SEDA paper explains this process very well (good read, worth your time), but basically what you gain by that is higher level of concurrently and better resource management, resource being CPU, disk, network etc.

So, to understand data flow in cassandra you first need to understand SEDA. Then you need to know which stages exist in cassandra and exactly does the data flow b/w them.

http://prettyprint.me/2010/05/02/understanding-cassandra-code-base/
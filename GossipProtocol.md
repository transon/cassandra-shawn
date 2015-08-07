The concept of gossip communication can be illustrated by the analogy of office workers spreading rumors. Ted comments to Sally that he believes Fred dyes his mustache. Sally tells Jill, while Ted repeats the idea to Sam. As people move around and repeat the rumor, the number of individuals who have heard the rumor roughly doubles with each "step". The doubling is an approximation that assumes employees communicate at a constant rate, and it doesn't account for gossiping twice to the same person. (Perhaps Ted tries to tell his story to Mark, only to find that Mark already heard it from Jill). Computer systems typically implement this type of protocol with a form of random "peer selection": with a given frequency, each machine picks another machine at random and shares any hot rumors.

The power of gossip lies in the robust spread of information. Even if Jill had trouble understanding Sally (perhaps she was whispering), she will probably run into someone else soon and can learn the news that way.

Expressing these ideas in more technical terms, a gossip protocol is one that satisfies the following conditions:

  * The core of the protocol involves periodic, pairwise, inter-process interactions.
  * The information exchanged during these interactions is of bounded size.
  * When agents interact, the state of at least one agent changes to reflect the state of the other. A gossip interaction does not occur when A pings B just to measure the response time, as this does not involve the transmittal of state between agents.
  * Reliable communication is not assumed.
  * The frequency of the interactions is low compared to typical message latencies so that the protocol costs are negligible.
  * There is some form of randomness in the peer selection. Peers might be selected from the full set of nodes or from a smaller set of "neighbors".


## Gossip (State Transfer Model) ##
In a state transfer model, each replica maintain a vector clock as well as a state version tree where each state is neither > or < among each other (based on vector clock comparison). In other words, the state version tree contains all the conflicting updates.
At query time, the client will attach its vector clock and the replica will send back a subset of the state tree which precedes the client's vector clock (this will provide monotonic read consistency). The client will then advance its vector clock by merging all the versions. This means the client is responsible to resolve the conflict of all these versions because when the client sends the update later, its vector clock will precede all these versions.

![http://cassandra-shawn.googlecode.com/files/stm_query_cr.png](http://cassandra-shawn.googlecode.com/files/stm_query_cr.png)

At update, the client will send its vector clock and the replica will check whether the client state precedes any of its existing version, if so, it will throw away the client's update.

![http://cassandra-shawn.googlecode.com/files/stm_update_cr.png](http://cassandra-shawn.googlecode.com/files/stm_update_cr.png)

Replicas also gossip among each other in the background and try to merge their version tree together.

![http://cassandra-shawn.googlecode.com/files/stm_update_rr.png](http://cassandra-shawn.googlecode.com/files/stm_update_rr.png)

## Gossip (Operation Transfer Model) ##

In an operation transfer approach, the sequence of applying the operations is very important. At the minimum causal order need to be maintained. Because of the ordering issue, each replica has to defer executing the operation until all the preceding operations has been executed. Therefore replicas save the operation request to a log file and exchange the log among each other and consolidate these operation logs to figure out the right sequence to apply the operations to their local store in an appropriate order.

"Causal order" means every replica will apply changes to the "causes" before apply changes to the "effect". "Total order" requires that every replica applies the operation in the same sequence.

In this model, each replica keeps a list of vector clock, Vi is the vector clock the replica itself and Vj is the vector clock when replica i receive replica j's gossip message. There is also a V-state that represent the vector clock of the last updated state.

When a query is submitted by the client, it will also send along its vector clock which reflect the client's view of the world. The replica will check if it has a view of the state that is later than the client's view.

![http://cassandra-shawn.googlecode.com/files/otm_query_cr.png](http://cassandra-shawn.googlecode.com/files/otm_query_cr.png)

When an update operation is received, the replica will buffer the update operation until it can be applied to the local state. Every submitted operation will be tag with 2 timestamp, V-client indicates the client's view when he is making the update request. V-@receive is the replica's view when it receives the submission.

This update operation request will be sitting in the queue until the replica has received all the other updates that this one depends on. This condition is reflected in the vector clock Vi when it is larger than V-client

![http://cassandra-shawn.googlecode.com/files/otm_update_cr.png](http://cassandra-shawn.googlecode.com/files/otm_update_cr.png)

On the background, different replicas exchange their log for the queued updates and update each other's vector clock. After the log exchange, each replica will check whether certain operation can be applied (when all the dependent operation has been received) and apply them accordingly. Notice that it is possible that multiple operations are ready for applying at the same time, the replica will sort these operation in causal order (by using the Vector clock comparison) and apply them in the right order.

![http://cassandra-shawn.googlecode.com/files/otm_update_rr.png](http://cassandra-shawn.googlecode.com/files/otm_update_rr.png)

The concurrent update problem at different replica can also happen. Which means there can be multiple valid sequences of operation. In order for different replica to apply concurrent update in the same order, we need a total ordering mechanism.

One approach is whoever do the update first acquire a monotonic sequence number and late comers follow the sequence. On the other hand, if the operation itself is commutative, then the order to apply the operations doesn't matter

After applying the update, the update operation cannot be immediately removed from the queue because the update may not be fully exchange to every replica yet. We continuously check the Vector clock of each replicas after log exchange and after we confirm than everyone has receive this update, then we'll remove it from the queue.

了解什么是Gossip协议，能够写一个简单的总结。
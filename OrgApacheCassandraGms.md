利用gossip + The Phi Accrual Failure Detector实现的这部分(**把你比我新的消息或者我没有的消息告诉我，我把比你新的或者你没有的消息告诉你**)。
下面这些文件描述信息只包含重要的部分！

  * VersionGenerator.java
```
包含一个原子操作的方法：getNextVersion
```

  * ApplicationState.java
```
定义了一个ApplicationState枚举类型
```

  * HeartBeatState.java
```
int generation_;
int version_;

除了上面这些属性以外，这个文件还包含一个类型为HeartBeatStateSerializer的serializer_属性，
它包含两个方法：
serialize： 按照下面的这个序列写入输出流

generation_, version_

deserialize: 按照serialize组织的序列解析
```

  * VersionedValue.java
```
public final int version;
public final String value;

VersionedValue同样包含一个类型为VersionedValueSerializer的serializer属性，
它包含两个方法：
serialize： 按照下面的这个序列写入输出流

value, version

deserialize: 按照serialize组织的序列解析
```

  * EndpointState.java
```
volatile HeartBeatState hbState_;
final Map<ApplicationState, VersionedValue> applicationState_;
/* fields below do not get serialized */
volatile long updateTimestamp_;
volatile boolean isAlive_;
volatile boolean isAGossiper_;
volatile boolean hasToken_;

除了上面这些属性以外，这个文件包含一个类型为EndpointStateSerializer的serializer_属性，
它包含两个方法：
serialize：按照下面这个序列写入输出流

HeartBeatState(generation_, version_)，size, [ApplicationState， VersionedValue(value, version)] ...

deserialize: 按照serialize的格式解析收到的message

```

  * GossipDigest.java
```
InetAddress endpoint_;
int generation_;
int maxVersion_;


VersionedValue同样包含一个类型为GossipDigestSerializer的serializer_属性，
它包含两个方法：
serialize： 按照下面的这个序列写入输出流

address length, address, generation_, maxVersion_

deserialize: 按照serialize组织的序列解析

```

  * IEndpointStateChangeSubscriber.java
```
/**
 * This is called by an instance of the IEndpointStateChangePublisher to notify
 * interested parties about changes in the the state associated with any endpoint.
 * For instance if node A figures there is a changes in state for an endpoint B
 * it notifies all interested parties of this change. It is upto to the registered
 * instance to decide what he does with this change. Not all modules maybe interested 
 * in all state changes.
 */

这个接口包含订阅者的callback方法：
onJoin
onChange
onAlive
onDead
onRemove
```

  * IFailureDetectionEventListener.java
```
/**
 * Implemented by the Gossiper to convict an endpoint
 * based on the PHI calculated by the Failure Detector on the inter-arrival
 * times of the heart beats.
 */
这个接口只包含一个方法：
convict
```

  * IFailureNotification.java
```
这个接口只包含下面两个方法
convict
revive
```

  * IFailureDetector.java
```
/**
 * An interface that provides an application with the ability
 * to query liveness information of a node in the cluster. It 
 * also exposes methods which help an application register callbacks
 * for notifications of liveness information of nodes.
 */

又是一个接口，包含下面这些方法，他们用于gossiper和failure detector之间互相沟通：
/**
* Failure Detector's knowledge of whether a node is up or
* down.
* 
* @param ep endpoint in question.
* @return true if UP and false if DOWN.
*/
public boolean isAlive(InetAddress ep);
    
/**
* This method is invoked by any entity wanting to interrogate the status of an endpoint. 
* In our case it would be the Gossiper. The Failure Detector will then calculate Phi and
* deem an endpoint as suspicious or alive as explained in the Hayashibara paper. 
* 
* param ep endpoint for which we interpret the inter arrival times.
*/
public void interpret(InetAddress ep);
    
/**
* This method is invoked by the receiver of the heartbeat. In our case it would be
* the Gossiper. Gossiper inform the Failure Detector on receipt of a heartbeat. The
* FailureDetector will then sample the arrival time as explained in the paper.
* 
* param ep endpoint being reported.
*/
public void report(InetAddress ep);

/**
* remove endpoint from failure detector
*/
public void remove(InetAddress ep);
    
/**
* Register interest for Failure Detector events. 
* @param listener implementation of an application provided IFailureDetectionEventListener 
*/
public void registerFailureDetectionEventListener(IFailureDetectionEventListener listener);
    
/**
* Un-register interest for Failure Detector events. 
* @param listener implementation of an application provided IFailureDetectionEventListener 
*/
public void unregisterFailureDetectionEventListener(IFailureDetectionEventListener listener);

```

  * FailureDetectorMBean.java
```
一个接口，包含下面这些方法：
public void dumpInterArrivalTimes();
public void setPhiConvictThreshold(int phi);
public int getPhiConvictThreshold();
public String getAllEndpointStates();
```

  * FailureDetector.java
```
/**
 * This FailureDetector is an implementation of the paper titled
 * "The Phi Accrual Failure Detector" by Hayashibara. 
 * Check the paper and the <i>IFailureDetector</i> interface for details.
 */

这个类实现了上面两个接口IFailureDetector, FailureDetectorMBean的方法

下面这两个属性，我觉得很重要：
private Map<InetAddress, ArrivalWindow> arrivalSamples_ = new Hashtable<InetAddress, ArrivalWindow>();
private List<IFailureDetectionEventListener> fdEvntListeners_ = new ArrayList<IFailureDetectionEventListener>();

其中ArrivalWindow包含两个fields:
private double tLast_ = 0L;
private BoundedStatsDeque arrivalIntervals_;

其中方法最重要当然就是：
public void report(InetAddress ep); // 数据采样
public void interpret(InetAddress ep); // 分析采样数据
public void registerFailureDetectionEventListener(IFailureDetectionEventListener listener); // gossiper注册自己的convict

```

  * Gossiper.java
```
/**
 * This module is responsible for Gossiping information for the local endpoint. This abstraction
 * maintains the list of live and dead endpoints. Periodically i.e. every 1 second this module
 * chooses a random node and initiates a round of Gossip with it. A round of Gossip involves 3
 * rounds of messaging. For instance if node A wants to initiate a round of Gossip with node B
 * it starts off by sending node B a GossipDigestSynMessage. Node B on receipt of this message
 * sends node A a GossipDigestAckMessage. On receipt of this message node A sends node B a
 * GossipDigestAck2Message which completes a round of Gossip. This module as and when it hears one
 * of the three above mentioned messages updates the Failure Detector with the liveness information.
 */

这个类实现了接口IFailureDetectionEventListener， 下面这些属性很关键：

/* subscribers for interest in EndpointState change */
private List<IEndpointStateChangeSubscriber> subscribers_ = new CopyOnWriteArrayList<IEndpointStateChangeSubscriber>();

/* live member set */
private Set<InetAddress> liveEndpoints_ = new ConcurrentSkipListSet<InetAddress>(inetcomparator);

/* unreachable member set */
private Set<InetAddress> unreachableEndpoints_ = new ConcurrentSkipListSet<InetAddress>(inetcomparator);

/* initial seeds for joining the cluster */
private Set<InetAddress> seeds_ = new ConcurrentSkipListSet<InetAddress>(inetcomparator);

/* map where key is the endpoint and value is the state associated with the endpoint */
Map<InetAddress, EndpointState> endpointStateMap_ = new ConcurrentHashMap<InetAddress, EndpointState>();

/* map where key is endpoint and value is timestamp when this endpoint was removed from
 * gossip. We will ignore any gossip regarding these endpoints for Streaming.RING_DELAY time
 * after removal to prevent nodes from falsely reincarnating during the time when removal
 * gossip gets propagated to all nodes */
 Map<InetAddress, Long> justRemovedEndpoints_ = new ConcurrentHashMap<InetAddress, Long>();

```

  * GossipDigestSynMessage.java
![http://cassandra-shawn.googlecode.com/files/gossipSynMessage.jpg](http://cassandra-shawn.googlecode.com/files/gossipSynMessage.jpg)

  * GossipDigestSynVerbHandler.java
  * GossipDigestAckMessage.java
![http://cassandra-shawn.googlecode.com/files/GossipAckMessage.jpg](http://cassandra-shawn.googlecode.com/files/GossipAckMessage.jpg)

  * GossipDigestAckVerbHandler.java
  * GossipDigestAck2Message.java
![http://cassandra-shawn.googlecode.com/files/GossipAck2Message.jpg](http://cassandra-shawn.googlecode.com/files/GossipAck2Message.jpg)

  * GossipDigestAck2VerbHandler.java

```

     Gossiper                                                       Gossipee
     --------                                                       ---------
         \                                                           
          \                                                            
           \             GossipDigestSynMessage                
            \----------------------------------------------->  GossipDigestSynVerbHandler
                        (GossipDigestList:request)                      /
                                                                       /
                                     GossipDigestAckMessage           /
GossipDigestAckVerbHandler  <----------------------------------------/
          \               (GossipDigestList:request, EtStateMap:response)
           \
            \          GossipDigestAck2Message
             \-------------------------------------------> GossipDigestAck2VerbHandler
                        (EtStateMap: response)
```

  * PureRandom.java
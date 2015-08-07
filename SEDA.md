通过paper对SEDA开始入门：

```
The fundamental unit of processing within SEDA is the stage. A stage is 
a self-contained application component consisting of an event handler, an 
incoming event queue, and a thread pool, as depicted in Figure 6. Each stage 
is managed by a controller that affects scheduling and thread allocation, 
as described below. Stage threads operate by pulling a batch of events off of 
the incoming event queue and invoking the application-supplied event handler. 
The event handler processes each batch of events, and dispatches zero or more 
events by enqueuing them on the event queues of other stages
```

**http://www.eecs.harvard.edu/~mdw/papers/seda-sosp01.pdf**

  * A SEDA design：

http://cassandra-shawn.googlecode.com/files/Figure%206.JPG

http://cassandra-shawn.googlecode.com/files/Figure%207.JPG

  * A SEDA prototype:

http://cassandra-shawn.googlecode.com/files/Figure%205.JPG

http://cassandra-shawn.googlecode.com/files/Figure%2010.JPG

2010/10/19：这个paper一共14页，计划用三天的时间读完。能够写一个简要的总结。
**Accrual failure detectors**: Failure detectors are traditionally based on a boolean
interaction model wherein processes can only either trust or suspect the processes
that they are monitoring. In contrast, we propose a novel abstraction, called accrual failure detector, whereby a failure monitor service outputs a value on a continuous scale rather than information of a boolean nature. Roughly speaking, this value captures the degree of confidence that a corresponding monitored process has crashed. If the process
actually crashes, the value is guaranteed to accrue over time and tend toward infinity, hence the name. It is then left to application processes to set an appropriate suspicion threshold according to their own quality-of-service requirements. A low threshold is prone to generate many wrong suspicions but ensures a quick detection in the event of a real crash. Conversely, a high threshold generates fewer mistakes but needs more time to detect actual crashes.

**Example**: Let us now illustrate the advantage of this approach with a simple example. Consider a distributed application in which one of the processes is designated as a master while all other processes play the role of workers. The master holds a list of jobs that needs to be computed and maintains a list of available worker processes. As long as jobs remain in its list, the master sends jobs to idle workers and collects the results after they have been computed. Assume now that some of the workers might crash (for simplicity, we assume that the master never crashes). Let some worker process Pw crash during the
execution; the master must be able to detect that Pw has crashed and take appropriate actions, otherwise the system might block forever. With accrual failure detectors, this could be realized as follows. When the confidence level reaches some low threshold, the master simply flags the worker process Pw and temporarily stops sending new jobs to Pw. Then, when reaching a moderate threshold, the master cancels all unfinished computations that were running on Pw and resubmit them to some other worker processes. Finally, when reaching a high threshold, the confidence that Pw has crashed is high, so the master removes Pw from its list of available workers and releases all corresponding resources. Using conventional failure detectors to implement such a simple behavior would be quite a challenge.

**Contribution**: In this paper, we present the abstraction of accrual failure detectors and describe an adaptive implementation called the ' failure detector. Briefly speaking, the ' failure detector works as follows. The protocol samples the arrival time of heartbeats and maintains a sliding window of the most recent samples. This window is used to estimate the arrival time of the next heartbeat, similarly to conventional adaptive failure detectors. The distribution of past samples is used as an approximation for the probabilistic distribution of future heartbeat messages. With this information, it is possible to compute
a value ' with a scale that changes dynamically to match recent network conditions. We have evaluated our failure detection scheme on a transcontinental link between Japan and Switzerland. Heartbeat messages were sent using the user datagram protocol (UDP) at a rate of about ten per second. The experiment ran uninterruptedly for a period of one week, gathering a total of nearly 6 million samples. Using these samples, we have analyzed the behavior of the ' failure detector, and compared it with traditional adaptive failure detectors. By providing exactly the same input to every failure detector, we could ensure the fairness of the comparison. The results show that the enhanced flexibility
provided by our approach does not induce any significant overhead.

  * Unreliable failure detectors

  * Quality of service of failure detectors

  * Heartbeat failure detectors
In this section, we present a brief overview of heartbeat-based implementations of failure detectors. Assume that processes have also access to some local physical clock giving them the ability to measure time. These clocks may or may not be synchronized. Using heartbeat messages is a common approach to implementing failure detectors. It works as follows
(see Fig. 1): process p—i.e., the monitored process—periodically sends a heartbeat message to process q, informing q that p is still alive. The period is called the heartbeat interval △i. Process q suspects process p if it fails to receive any heartbeat message from p for a period of time determined by a timeout △to, with △to ≧ △i. A third value of importance is the network transmission delay of messages. For convenience,
we denote by △tr the average transmission time experienced by messages. In the conventional implementation of heartbeat-based failure detection protocols, the timeout △to is fixed as a constant value. Upon receiving a heartbeat, process q waits for the next heartbeat for at most △to units of time, after which it begins to suspect process p if no new heartbeat has been received. Obviously, the choice of a timeout value must be larger than △i, and is dictated by the following tradeoff. If the timeout (△to) is short, crashes are detected quickly but the likeliness of wrong suspicions is high. Conversely, if the timeout is long, wrong suspicions become less frequent, but this comes at the expense of detection time. An alternative implementation of heartbeat failure detectors sets a timeout based on the transmission time of the heartbeat. The advantage of this approach is that the maximal detection time is bounded, but its drawback is that it relies on clocks with negligible drift² and a shared knowledge of the heartbeat interval △i. This last point can become a problem in practice, when the regularity of the sending of heartbeats cannot be ensured and a short interval makes the timing inaccuracies due to operating system scheduling take more importance (i.e., the actual interval differs from the target one as a result).

```

 <发送heartbeat给q的时间间隔>                   <已经过了这么长时间没收到p的heartbeat了>
           ∕                                                        /
          ∕                                                        /
         ∕                                                        /
      p(△i)                  heartbeat                         q(△to)
__________________        ==================>             _____________________
monitored process                 △tr                      monitoring process
                                   \
                                    \
                                     \
                            <网络传送hearbeat的时间>
```

http://cassandra-shawn.googlecode.com/files/heartbeat_failure_detection.JPG


q在tstart + △t时刻收到p的第一个heartbeat消息，经过△i间隔，p发送另外一个heartbeat给q,q在△to时间间隔内收到了这个heartbeat,p发送另外一个heartbeat给q，q这次等了很长时间（超过了△to，可能因为网络传递延迟等原因，这次等的久了点）才收到这个heartbeat消息，接收q等了许久（超过了△to）再也没收到p的heartbeat消息了，所以q认为p挂掉了。。。


  * Adaptive failure detectors

  * Accrual failure detectors

The principle of accrual failure detectors is simple. Instead of outputting information of a boolean nature, accrual failure detectors output suspicion information on a continuous scale. Roughly speaking, the higher the value, the higher the chance that the monitored process has crashed.

A. Architecture overview
Conceptually, the implementation of failure detectors on the receiving side can be decomposed into
three basic parts as follows.
  1. Monitoring. The failure detector gathers information from other processes, usually through the network, such as heartbeat arrivals or query-response delays.
  1. Interpretation. Monitoring information is used and interpreted, for instance to decide that a process should be suspected.
  1. Action. Actions are executed as a response to triggered suspicions. This is normally done within applications.

The main difference between traditional failure detectors and accrual failure detectors is which component of the system does what part of failure detection. In traditional timeout-based implementations of failure detectors, the monitoring and interpretation
parts are combined within the failure detector (see Fig. 2). The output of the failure detector is of boolean nature; trust or suspect. An elapsing timeout is equated to suspecting the monitored process, that is, the monitoring information is already being interpreted. Applications cannot do any interpretation, and thus are left with what to do with the suspicion. Unfortunately, suspicion tradeoffs largely depend on the nature
of the triggered action, as well as its cost in terms of performance or resource usage.
In contrast, accrual failure detectors provide a lower-level abstraction that avoids the interpretation of monitoring information (see Fig. 3). Some value is associated with each process that represents a suspicion level. This value is then left for the applications to interpret. For instance, by setting an appropriate threshold, applications can trigger suspicions and perform appropriate actions. Alternatively, applications can directly use the value output by the accrual failure detector as a parameter to their actions. Considering the example of master/worker described in the introduction, the master could decide to allocate the most urgent jobs only to worker processes with a low suspicion level.

http://cassandra-shawn.googlecode.com/files/structure_between_traditional_and_accural_failure.JPG

B. Implementation

http://cassandra-shawn.googlecode.com/files/accural_failure_cal3.JPG

http://cassandra-shawn.googlecode.com/files/accural_failure_cal1.JPG

http://cassandra-shawn.googlecode.com/files/accural_failure_cal2.JPG
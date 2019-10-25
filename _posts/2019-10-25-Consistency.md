---
layout: post
title: Notes on Consistency
published: true
category:
    - Distributed Systems
    - DataBase
    - Consistency
---

* Consistency mean a read from a system should return the latest write.
* This can be achieved using various kinds of locking mechanism in case of a single node multi-threaded mechanism. All threads can be synchronized because they share the same clock.
* In case of distributed system, where a global clock is not available, we need to figure out what the latest write was, to make the read consistent.

### Strict Consistency
* Strict consistency is the strongest consistency model. 
* Under this model, a write to a variable by any processor needs to be seen instantaneously by all processors.
![Strict consistency]({{ site.url }}/images/Strict-Consistency.PNG)

### Sequential Consistency
* `The result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.` - Leslie Lamport
* What the above statement means is, that, for each thread, operations need to happen in program order. A thread `T1` with operations `A1`, `B1` and `C1` will have to execute these operations in that sequence. Across multiple processors, the sequence is undefined. Thread `T2` with operations `A2` and `B2`, can be scheduled in anyway with respect to thread `T1` as long as `A2` and `B2` maintain their mutual order between themselves. `Sequential consistency gaurantees Program Order.`
* In the image below, write from P1 and P2 may be scheduled in any order amongst themselves, but once an ordering has been decided, P3 and P4 should see them in the same order.
![Sequential consistency]({{ site.url }}/images/Sequential-Consistency.PNG)

### Causal Consistency
* Causal consistency can be defined as a model that captures the causal relationships between operations in the system and guarantees that each process can observe those causally related operations in common causal order.
*  If there is an operation or event A that causes another operation B, then causal consistency provides an assurance that each other process of the system observes operation A before observing operation B.
* Causal consistency is a weaker consistency model than Sequential consistency, because Sequential consistency expects all processes to observe not just causally related writes but all write operations in common order.
* Causal consistency is stronger than __PRAM consistency__, which requires only the `write operations that are done by a single process to be observed in common order by each other process`. 
* *It follows that when a system is sequentially consistent, it is also causally consistent. Additionally, causal consistency implies PRAM consistency, but not vice versa.
* Write operation `W(x)2` is caused by an earlier write `W(x)1`, and it means that these two writes `W(x)1` and `W(x)2` are causally related as process `P2` observes, reads, the earlier write `W(x)1` that is done by process `P1`. Thus, every other process observes `W(x)1` first before observing `W(x)2`. 
* Also, another thing to notice is that the two write operations `W(x)2` and `W(x)3`, with no intervening read operations, are concurrent since processes `P3` and `P4` observe, read, them in different orders. 
![Causal consistency]({{ site.url }}/images/Causal-Consistency.PNG)
* There are four different session guarantees that are ensured and guaranteed by the causal consistency model. Those guarantees are summarized below:
    
##### Read Your Writes  
* This means that preceding write operations are indicated and reflected by the following read operations. 
* The Read Your Writes guarantee ensures that whenever a client reads a value after updating it, it sees that value
![Read Your Writes]({{ site.url }}/images/Read-Your-Writes.PNG)

##### Monotonic Reads
* This implies that an up-to-date increasing set of write operations is guaranteed to be indicated by later read operations.
* A non-monotonic read anomaly occurs within a session when a read of some object returns a given version, and a subsequent read of the same object returns an earlier version.
![Monotonic Reads]({{ site.url }}/images/Monotonic-Read.PNG)

##### Writes Follow Reads
* This provides an assurance that write operations follow and come after reads by which they are influenced.
* A non-monotonic transaction anomaly occurs if a session observes the effect of transaction T1 and then commits T2, and another session sees the effects of T2 but not T1. The Writes Follow Reads guarantee prevents these anomalies.
![Writes-Follow-Reads]({{ site.url }}/images/Writes-Follow-Reads.PNG)

##### Monotonic Writes
* This guarantees that write operations must go after other writes that reasonably should precede them.
* A non-monotonic write anomaly occurs if a session’s writes become visible out of order.
![Monotonic Writes]({{ site.url }}/images/Monotonic-Write.PNG)

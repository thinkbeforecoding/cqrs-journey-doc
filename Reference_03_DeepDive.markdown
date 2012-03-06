## Chapter 4

# A CQRS/ES Deep Dive (Chapter Title)

# The Domain Model  

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Aggregate Roots, Bounded Contexts etc.
  </span>
</div> 

# Commands and CommandHandlers 

# Events and EventHandlers 

# Embracing Eventual Consistency 

Maintaining the consistency of business data is a key requirement in all 
enterprise systems. One of the first things that many developers learn 
in relation to database systems is the ACID properties of transactions: 
transactions must ensure that the stored data is consistent as well 
transactions being atomic, isolated, and durable. Developers also become 
familiar with complex concepts such as pessimistic and optimistic 
concurrency, and their performance characteristics in particular 
scenarios. They may also need to understand the different isolation 
levels of transactions: serializable, repeatable reads, read committed, 
and read uncommitted. 

In a distributed computer system, there are some additional factors that 
are relevant to consistency. The CAP theorem states that it is 
impossible for a distributed computer system to provide the following 
three guarantees simultaneously: 

1. Consistency. A guarantee that all the nodes in the system see the 
   same data at the same time. 
2. Availability. A guarantee that the system can continue to operate
   even if a node is unavailable. 
3. Partition tolerance. A guarantee that the system continues to operate
   despite the nodes being unable to communicate. 

For more information about the [CAP theorem][captheorem], see CAP 
theorem on Wikipedia. 

The concept of *eventual consistency* offers to way to make these three 
guarantees simultaneously by relaxing the guarantee of consistency. In 
the CAP theorem, the consistency guarantee specifies that all the nodes 
should see the same data *at the same time*; if instead, we state that 
all the nodes will eventually see the same data, we can make the 
consistency guarantee compatible with the other two. Another way of 
viewing this is to say that we will accept that at any given time, some 
of the data seen by users of the system could be stale. For many 
business scenarios, this turns out to be perfectly acceptable: a 
business user will accept that the information they are seeing on a 
screen may be a few seconds, or even minutes out of date. Depending on 
the details of the scenario, the business user can refresh the display a 
bit later on to see what has changed, or simply accepts that what they 
see is always slightly out of date. There are some scenarios where this 
delay is unacceptable, but they tend to be the exception rather than the 
rule. 

# Eventual Consistency and CQRS 

How does the concept of eventual consistency relate to the CQRS pattern? 
A typical implementation of the CQRS pattern is a distributed system 
made up of one node for the write-side, and one or more nodes for the 
read-side. Your implementation must provide some mechanism for 
synchronizing data between these two sides. This is not a complex 
synchronization task because all of the changes take place on the 
write-side, so the synchronization process only needs to push changes 
from the write-side to the read-side. 

If you decide that the two sides must always be consistent, then you 
will need to introduce a distributed transaction that spans both sides 
as shown in figure 1. 

![Figure 1][fig1]

**Using a distributed transaction to maintain consistency**

The problems that may result from this approach relate to performance 
and availability. Firstly, both sides will need to hold locks until both 
sides are ready to commit, in other words the transaction can only 
complete as fast as the slowest participant can. 

This transaction may include more than two participants. If we are 
scaling the read-side by adding multiple instances, the transaction must 
span all of those instances. 

Secondly, if one node fails for any reason or does not complete the 
transaction, the transaction cannot complete. In terms of the CAP 
theorem, by guaranteeing consistency, we cannot guarantee the 
availability of the system. 

If you decide to relax your consistency constraint and specify that your 
read-side only needs to be eventually consistent with the write-side, 
you can change the scope of your transaction. Figure 2 shows how you can 
make the read-side eventually consistent with the write-side by using a 
reliable messaging transport to propagate the changes. 

![Figure 2][fig2]

**Using a reliable message transport**

In this example, you can see that there is still a transaction. The 
scope of this transaction includes saving the changes to the data store 
on the write-side, and placing a copy of the change onto the queue that 
pushes the change to the read-side. 

This solution does not suffer from the potential performance problems 
that you saw in the original solution if you assume that the messaging 
infrastructure allows you to quickly add messages to a queue. This 
solution is also no longer dependent on all of the read-side nodes being 
constantly available because the queue acts as a buffer for the messages 
addressed to the read-side nodes. 

> **Note:** In practice, the messaging infrastructure is likely to use a 
  publish/subscribe topology rather than a queue to enable multiple 
  read-side nodes to receive the messages. 

This third example in figure z shows a way that you can do away with the 
need for a distributed transaction. 

![Figure 3][fig3]

**No distributed transactions**

This example depends on functionality in the write-side data store: it 
must be able to send a message in response to every update that the 
write-side model makes to the data. This approach lends itself 
particularly well to the scenario where you combine CQRS with event 
sourcing. If the event store can send a copy of every event that it 
saves onto a message queue, then you can make the read-side eventually 
consistent by using this infrastructure feature. 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	[Add a reference to an event store that offers this functionality.]
  </span>
</div>  

Queries 

# Optimizing the Read Side 

# Optimizing the Write Side 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	parallelizing command handling, events, etc.<br/>
	Snapshots in the event store
  </span>
</div>

# Messaging 

## Messaging and CQRS 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Discuss role of messaging infrastructure and CQRS. Transport for commands and events. Topologies: queues and pub/sub.
  </span>
</div> 

## Messaging Considerations 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	Problems that arise with out of order messages, duplicate messages, lost messages. Strategies for dealing with these issues. 
	Need to discuss issues of dropped, multiple, out of order delivery here. <br />
	Multiple delivery of messages or lost messages. This is a general messaging issueIt is possible that due to a problem in the messaging infrastructureIf it is possible that the read-side could receive multiple copies of a single event, for example because of a problem in the messaging infrastructure, you must ensure that your system state remains correct. For example, an individual subscriber could receive multiple copies introducing a temporary error that you could fix by replaying the events. For example, if a reservation event is delivered twice, then you may end up with an inaccurate figure for the total number of bookings for a conference. Designing event messages to be idempotent or assigning messages unique identifiers are possible strategies to help address this problem. Similar problems can result from subscribers not receiving messages. <br />
	Out of order delivery. In some scenarios, the specific ordering of a set of event messages may be significant. Again, it may be that your messaging infrastructure does not guarantee ordered delivery. Assigning sequence numbers to messages, using batches, or using sagas are all possible strategies for addressing this problem. The chapter "A CQRS/ES Deep Dive" includes a discussion of sagas. <br />
  </span>
</div> 


## Messaging Infrastructure 

<div style="margin-left:20px;margin-right:20px;">
  <span style="background-color:yellow;">
    <b>Comment [DRB]:</b>
	What to expect from an infrastructure messaging service. <br />
	Reference Windows Azure Service Bus section below, but mention other products. 
  </span>
</div>

# Task-based UIs 

# Taking Advantage of Windows Azure

In the chapter "[Introducing the Command Query Responsibility 
Segregation Pattern][r_chapter2]," we suggested that the motivations for 
hosting an application in the cloud were similar to the motivations for 
implementing the CQRS pattern: scalability, elasticity, and agility. 
This section describes in more detail how a CQRS implementation might 
use some of specific features of the Windows Azure platform to provide 
some of the infrastructure that you typically need when you implement 
the CQRS pattern. 

## Scaling Out Using Multiple Role Instances 

When you deploy an application to Windows Azure, you deploy the 
application to roles in your Windows Azure environment; a Windows Azure 
application typically consists of multiple roles. Each role has 
different code and performs a different function within the application. 
In CQRS terms, you might have one role for the implementation of the 
write-side model, one role for the implementation of the read-side 
model, and another role for the UI elements of the application. 

After you deploy the roles that make up your application to Windows 
Azure, you can specify (and change dynamically) the number of running 
instances of each role. By adjusting the number of running instances of 
each role, you can elastically scale your application in response to 
changes in levels of activity. One of the motivations for using the CQRS 
pattern is the ability to scale the read-side and the write-side 
independently given their typically different usage patterns. For 
information about how to automatically scale roles in Windows Azure, see 
"[The Autoscaling Application Block][aab]" on MSDN. 

## Implementing an Event Store Using Windows Azure Table Storage 

## Implementing a Messaging Infrastructure Using the Windows Azure Service Bus 


[r_chapter1]:     Reference_01_CQRSContext.markdown
[r_chapter2]:     Reference_02_CQRSIntroduction.markdown
[r_chapter4]:     Reference_04_DeepDive.markdown


[captheorem]:		http://en.wikipedia.org/wiki/CAP_theorem
[aab]:				http://msdn.microsoft.com/en-us/library/hh680892(PandP.50).aspx


[fig1]:           images/Reference_04_Consistency_01.png?raw=true
[fig2]:           images/Reference_04_Consistency_02.png?raw=true
[fig3]:           images/Reference_04_Consistency_03.png?raw=true
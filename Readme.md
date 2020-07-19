# Kafka notes

## The problem Kafka tries to solve is:

If you have 4 'main' services, and 6 'sub' services, you need to write 24 integrations all both the 'main' and the 'sub' level.

## Why Apache Kafka

- Distributed, resilient architecture that is fault tolerant
- Horizontal scalability (can scale 100s of brokers and millions of messages per second)
- High performance (latency of less than 10ms)
- Used by the big dogs (Netflix, LinkedIN, Uber, Walmart)

## Usecases

- Netflix uses kafka to apply recommendations in real-time
- Uber uses it to gather user, taxi and trip data in real-time to compute forecast demand and computer surge-pricing in real-time
- LinkedIN uses it to prevent spam, collect user interactions and make better connection recommendations.

**Remember** - Kafka is only used as a transportation mechanism! You still need to write applications to make things work.

## My objectives

- Ability to use Kafka on the CLI

## Topics, partitions and offsets

**Topic**

Topics are a particular stream of data.

- A good way to think about a topic is a table in a DB.
- You can have as many topics as you want
- A topic is identified by its name

Topics are split in _partitions_

- Each partition is ordered (starting at 0)
- Each message within a partition gets an incremental id called _offset_ (also starting at 0)

When you create a topic, you need to specify how many partitions you want. You can change this later on.

Now, when you start getting messages into the topic, the first one going into partition 0, starts with _id_ or _offset_ 0.

![1.png](./images/1.png)

In this case above, we have 3 partitions (0, 1 and 2).

The first message into partition 0 will have offset 0, then 1, 2 and so on..

To refer to a message, we'll be referring to it as `Partition 0, offset 0` etc.

**Topic example use case**

![2.png](./images/2.png)

- You have a fleet of trucks. Each truck reports its GPS position to Kafka.
- We can have a topic `trucks_gps` as the topic that contains the position of all trucks in the fleet.
- Each truck will send a message to Kafka every 20 seconds.
- Each message will contain the truck ID and the truck position (lat long).

**Note**: _The location dashboard & notification service in the diagram above are our consumers_. _More on that later_.

Our `trucks_gps` topic will have an arbitrary number of partitions (let's say 10).

A few gotchas:

- Offsets only have a meaning if you know what partition its in (Offset 1 in partition 0 is totally a different message to Offset 1 in partition 1 for example)
- Order is only guaranteed within a partition. For eg: we can guarantee that offset 5 in partition 0 has been written before offset 6, 7 and 8. BUT we cannot offset 5 in partition 1 was written before offset 6 in partition 0.
- Data is only kept for a limited period of time (by default for 1 week).
  The data inside the offsets will be deleted, but it's immutable. You cannot update it and even once it's deleted, you will have to add it to another offset.
- Data is assigned randomly to a partition unless a key is provided. So if you try to add a message, it will be added to partition 0, 1 or 2 and wew can't control that.

# Brokers

- A Kafka cluster is composed of multiple brokers (or servers).

- Each broker is identified by an ID (which is always a number)

- Each broker will contain certain topic partitions, so it will have some of the data but not all the data because Kafka is distributed.

- When you're connected to a particular Kafka broker (called a bootstrap broker), you will be connected to the entire cluster.

- A good number to get started is 3 brokers, but some big clusters have over 100 brokers.

# Brokers & Topics

Let's say we have 3 brokers (101, 102 & 103):

![3.png](./images/2.png)

Let's say we have a topic called `Topic-A` and it has **3 partitions**

Our topic, on creation, is distributed amongst our brokers by Kafka. Note that there is no relation between our partition number and broker ID.

![4.png](./images/4.png)

Say we have another topic `Topic-B`, and this topic only has **2 partitions**.

![5.png](./images/5.png)

Data is distributed and Broker 103 doesn't have any topic B data.

## Topic replication factor

Kafka is a distributed system. In a distributed system, we need replication for resiliency. If a machine goes down, the system cannot just stop serving up data.

- Topics should have a replication factor > 1 (usually between 2 & 3, and 3 being the gold standard).
- When you create a topic, you want it to be replicated. If a broker is down, then another broker can serve the data you need.

Let's consider this in a new example:

Our topic here is called `Topic-A`, and has two partitions (`partition 0` and `partition 1`).

![6.png](./images/6.png)

We also have a replication factor of 2. Which will mean our partitions will be replicated twice, like so:

![6.png](./images/7.png)

Let's look at what happens if we lose Broker 102 in our example:

![8.png](./images/8.png)

Even with broker 102 down, we still have access to all our data!

## Partition leaders

The golden rule is:

- At any given time, only ONE broker can be a leader for a given partition.
  Only that leader can receive and serve data for a partition.
- The other brokers will only passively sync the data for that partition.
- Each partition has one leader and multiple in-sync replicas (ISR).

For partition 0, broker 101 is the leader and broker 102 is the ISR.
For partition 1, broker 102 is the leader and broker 103 is the ISR.

The system that decides leaders and ISRs is called `Zookeeper`. If a broker goes down, there's an election to decide the new leader. Once the broker that went down comes back up - it will try to become the leader again after syncing the data.

# Producers

- Producers write data to topics (which in turn is made of partitions)
- Producers automatically know to which broker and partition to write to
- In case of broker failures, producers will automatically recover (this is part of kafka).

Here's the sequence diagram:

![9.png](./images/9.png)

Our producer will send data to our brokers. Basically, if the data does not have a key - that data will be sent round robin to broker 101, 102 and 103.

The producer automatically load balances, i.e. it sends a bit to 101, a bit to 102 and a bit to 103.

If we take this exact same topic, let's look at how the producer does it's job.

Producers can choose to receive ackowledgement of data writes.

```
acks=0: Producer won't wait for ackowledgement of data write
acks=1: Producer will wait for leader acknowledgment
acks=all: Leader + replicas send acknowledgement of data write
```

## Message keys for producers

- Producers can choose to send a key with messages (can be a string, number etc)
- If key=null, data is sent round robin (broker 101, then 102, then 103).
- If a key is sent to the producer, then all messages for that key will always go to the same partition
- A key is basically sent if you need message ordering for a specific field (eg: truck_id)

### Truck example

If we go back to the earlier trucking example (1000 trucks in a fleet, all sending their lattitude and longitude every 20 seconds along with the truck ID):

- For a given truck, we want the GPS data for that truck (say truck id: 123) to be in order
- We'll choose this truck id 123 as the key when we send messages to the producer. 
- When a truck id is present, this data will always go to the same partition (for eg: partition 0)
- This mechanism of a certain key going to a certain partition is called as key hashing. 

# Consumers

- Consumers read data from a topic (identified by name)
- Consumers know which broker to read from
- In case of broker failures, consumers know how to recover
- Data is read in order within each partitions

![10.png](./images/10.png)

In the diagram above, the consumer will read data at offset 0 before reading the data at offset 3, before reading the data at offset 10, and so on. 

**But there's no ordering between two separate partitions**

![11.png](./images/11.png)

Again, there's no guarantee of order between two separate partitions. It might read offset 3 on partition 1 before reading offset 2 on partition 0 (for example).

## Consumer Groups

- A consumer can be a node app (for example). 
- Consumers read data in consumer groups.
- Each consumer within a group reads from exclusive partitions.
- If you have more consumers than partitions, some consumers will be inactive. 










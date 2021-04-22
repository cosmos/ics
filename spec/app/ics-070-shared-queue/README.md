# Queue Module

## Introduction

This document presents a technical specification of the **queue module**.
The queue module contains a logic necessary for implementing shared FIFO queue among two parties (i.e., two blockchains).
Therefore, in an implementation of the FIFO queue we are interested in, the following entities take place:
  - Two blockchains: Each validator of each blockchain implements its queue module.
  We say that one blockchain is the parent blockchain, whereas the other is the baby blockchain.
  - IBC communication: There exists an IBC communication among two aforementioned blockchains.
  Blockchains communicate exclusively using the IBC channels among them.

We present two versions of the queue module.
The versions provide slightly different interface and guarantees, as we describe in the following subsection.

### Shared FIFO Queue Specification

Our shared FIFO queue is a concurrent object.
We now define both versions of our queue.

#### Sequential Consistent FIFO Queue

The first "version" of the queue - which is denote by Sequential Consistent FIFO queue (SCQ) - exposes the following interface:
- *enqueue(x)* operation: Operation that enqueues the element *x* to the queue.
- *dequeue()* operation: Operation that dequeues the first element of the queue and returns that element.
- *peak()* operation: Operation that returns the first element of the queue without removing it.
- *contains(x)* operation: Operation that returns whether the element *x* belongs to the queue.

We set the following constraints:
- Enqueue and contains operations are invoked solely by the parent blockchain.
- Dequeue and peak operations are invoked solely by the baby blockchain.

Lastly, our implementation satisfies the **sequential consistency** correctness criterium:
The result of any execution is the same as if the operations of all the processes were executed in some sequential order, and the operations of each individual process appear in this sequence in the order specified by its protocol.

#### Causally Consistent FIFO Queue

We denote the second version of our FIFO queue by Causally Consistent FIFO Queue (CCQ), which exposes the following interface:
- *enqueue(x)* operation: Operation that enqueues the element *x* to the queue.
- *dequeue()* operation: Operation that dequeues the first element of the queue and returns that element.
- *read()* operation: Operation that returns the "content" of the entire queue.

We introduce the following constraints:
- Enqueue operations are invoked solely by the parent blockchain.
- Dequeue operations are invoked solely by the baby blockchain.

Note the difference between causally and sequentially consistent FIFO queue in their interface.
The SCQ exposes the peak and contains operations (that are invoked solely by the baby and parent blockchain, respectivelly), whereas CCQ exposes the read operation (that can be invoked by both blockchains).
Note that read operation of the CCQ could be used to implement the peak and contains operations (i.e., the read operation could be reduced to the peak and contains operations).
However, we show that the approach used for implementing SCQ, which is sequentially consistent, **cannot** be used for implementing sequentially consistent queue with the read operation.

Thus, CCQ satisfies the **causal consistency** correctness criterium.
Let us first define the causal precedence relation:
Consider operations A and B.
We say that A *causally precedes* B (we write "A --> B") if and only if:
1) A and B are invoked by the same blockchain and A is invoked before B, or
2) A is a write operation and the blockchain that invokes B has observed A before invoking B, or
3) There exists an operation C such that A --> C and C --> B.  
Moreover, operations A and B are *concurrent* if neither causally precedes the other.

Finally, we are able to define the causal consistency correctness criterium.
Operations that are related by the causal precedence relation are observed by all parties (i.e., blockchains) in their causal precedence order.
Note that the definition allows concurrent operations to be seen in different order by different blockchains.

### Closer Look at the IBC Channels

An IBC channel assumes two parties (the respective blockchains) involved in the communication. However, it also assumes a relayer which handles message transmissions between the two blockchains. The relayer carries a central responsibility in ensuring communication between the two parties through the channel.

A relayer intermediates communication between the two blockchains. Each blockchain exposes an API comprising read, write, as well as a queue (FIFO) functionality. So there are two parts to the communication API:

- a read/write store: The read/write store holds the entire state of the chain. Each module can write to this store.
- a queue of datagrams (packets): Each module can dequeue datagrams stored in this queue and a relayer can queue to this queue.

## High-Level Design of the Shared FIFO Queue

In this section, we provide an intuition behind our protocols.
Since both SCQ and CCQ are implemented in the fairly similar fashion, we do not distinguish the two here.

First, each blockchain maintains a local copy Q of our FIFO shared queue.

Let us first explain how we implement the write operations (i.e., enqueue and dequeue operations).
Firstly, the blockchain that has invoked a write operation executes the operation on its local copy.
Then, it informs the other blockchain about the operation using the IBC channel between the two blockchains.
Lastly, the operation completes once the invoking blockchain receives an IBC acknowledgment for the sent packet.

As for the read operations (peak and contains in SCQ, or read in CCQ), an invoking blockchain simply performs the read operation on its local copy.
Note that no communication between blockchain occurs in case of read operations.

## Data Structures

We devote this section to defining the data structures used to represent the states of both parent and baby blockchain, as well as the packets exchanged by the two blockchains.

### Application data

#### The parent blockchain

- Q: Local copy of the shared FIFO queue.
- Number: Map <Element, Integer> that takes note of how many instances of a specific element are present in the local copy of Q (used exclusively in the implementation of contains operations in SCQ).
- Outcoming data store: "Part" of the blockchain observable for the relayer. Namely, each packet written in outcoming data store will be relayed by the relayer.

#### The baby blockchain

- Q: Local copy of the shared FIFO queue.
- Outcoming data store: "Part" of the blockchain observable for the relayer. Namely, each packet written in outcoming data store will be relayed by the relayer.

### Packet data

- OperationPacket: Packet sent by an invoking blockchain to the other blockchain and encapsulates an invoked operation.
More specifically, the packet has three parameters: 1) a unique identifier of the operation (can be blockchain id + sequence number), 2) type of the operation, and 3) a parameter (enqueued element in the case of an enqueue operation or dequeued element in the case of a dequeue operation).

*Remark:* There exists the default acknowledgment packet for the OperationPacket.

## Implementation

### Port & channel setup

The `setup` function must be called exactly once when the module is created
to bind to the appropriate port.

```golang
func setup() {
  capability = routingModule.bindPort("cross-chain staking", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
  claimCapability("port", capability)
}
```

Once the `setup` function has been called, channels can be created through the IBC routing module
between instances of the cross-chain staking modules on mother and daughter chains.

##### Channel lifecycle management

Mother and daughter chains accept new channels from any module on another machine, if and only if:

- The channel being created is ordered.
- The version string is `icsXXX`.

```golang
func onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // assert that version is "icsXXX"
  abortTransactionUnless(version === "icsXXX")
}
```

```golang
func onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // assert that version is "icsXXX"
  abortTransactionUnless(version === "icsXXX")
  abortTransactionUnless(counterpartyVersion === "icsXXX")
}
```

```golang
func onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // port has already been validated
  // assert that version is "icsXXX"
  abortTransactionUnless(version === "icsXXX")
}
```

```golang
func onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated, version has already been validated
}
```

```golang
func onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // the channel is closing, do we need to punish?
}
```

```golang
func onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // the channel is closed, do we need to punish?
}
```

### The parent blockchain

```golang
func enqueue(x: Element) {
  // enqueue x locally
  Q.enqueue(x)

  // increase the number of occurrences of x; just for SCQ
  Number[x] = Number[x] + 1

  // create the OperationPacket packet
  OperationPacket packet = OperationPacket{uniqueOperationId, "enqueue", x}

  // obtain the destination port of the consumer
  destPort = getPort(babyChainId)

  // send the packet
  handler.sendPacket(Packet{timeoutHeight, timeoutTimestamp, destPort, destChannel, sourcePort, sourceChannel, packet}, getCapability("port"))
}
```

- Expected precondition
  - None
- Expected postcondition
  - The element x is enqueued in the local copy of the queue.
  - The number of occurrences of x is incremented.
  - The OperationPacket is created
- Error condition
  - If the precondition is violated

```golang
func contains(x: Element) {
  return Number[x] > 0
}
```

- Expected precondition
  - None
- Expected postcondition
  - None
- Error condition
  - If the precondition is violated

```golang
func read() {
  return Q
}
``` 

- Expected precondition
  - None
- Expected postcondition
  - None
- Error condition
  - If the precondition is violated

```golang
func onRecvPacket(packet: Packet) {
  // the packet is of OperationPacket type
  assert(packet.type = OperationPacket)

  // the packet is for a dequeue operation
  assert(packet.operation = "dequeue")

  // get x
  x = packet.value

  // remove the first appearance of x from the local copy
  Q.remove(x)

  // decrease the number of occurrences of x
  Number[x] = Number[x] - 1

  // construct the default acknowledgment
  ack = defaultAck(OperationPacket)
  return ack
}
```

- Expected precondition
  - The packet is sent to the parent by the baby blockchain
  - The packet is of *OperationPacket* type
  - The packet is for a dequeue operation
- Expected postcondition
  - The element x is removed from the local copy
  - The number of occurrences of x is decremented
- Error condition
  - If the precondition is violated

### The baby blockchain

```golang
func dequeue() {
  // dequeue locally
  x = Q.dequeue()

  // create the OperationPacket packet
  OperationPacket packet = OperationPacket{uniqueOperationId, "dequeue", x}

  // obtain the destination port of the consumer
  destPort = getPort(parentChainId)

  // send the packet
  handler.sendPacket(Packet{timeoutHeight, timeoutTimestamp, destPort, destChannel, sourcePort, sourceChannel, packet}, getCapability("port"))

  return x
}
```

- Expected precondition
  - None
- Expected postcondition
  - The element x is dequeued from the local copy of the queue.
  - The number of occurrences of x is decremented.
  - The OperationPacket is created
- Error condition
  - If the precondition is violated

```golang
func peak() {
  return Q.peak()
}
```

- Expected precondition
  - None
- Expected postcondition
  - None
- Error condition
  - If the precondition is violated

```golang
func read() {
  return Q
}
``` 

- Expected precondition
  - None
- Expected postcondition
  - None
- Error condition
  - If the precondition is violated

```golang
func onRecvPacket(packet: Packet) {
  // the packet is of OperationPacket type
  assert(packet.type = OperationPacket)

  // the packet is for an enqueue operation
  assert(packet.operation = "enqueue")

  // get x
  x = packet.value

  // enqueue x
  Q.enqueue(x)

  // construct the default acknowledgment
  ack = defaultAck(OperationPacket)
  return ack
}
```

- Expected precondition
  - The packet is sent to the parent by the baby blockchain
  - The packet is of *OperationPacket* type
- Expected postcondition
  - The element x is enqueued to the local copy
- Error condition
  - If the precondition is violated

## Correctness arguments

### SCQ

TODO

### CCQ

#### CCQ is **not** sequentially consistent

First, we show why CCQ we presented is not sequentially consistent.
Consider the following example:
The parent blockchain issues enqueue(1), enqueue(2), enqueue(3) and read() = {1, 2, 3}.
Moreover, the baby blockchain issues dequeue() = 1 and read() = {2}.

Let us show that the aforementioned operations can indeed return the presented values.
The parent blockchain issues the three enqueue operations and the read operation is invoked before the parent blockchain observes the dequeue() = 1 operation.
Moreover, the baby blockchain issues the *dequeue() = 1* operation after it observes the enqueue(1) operation and it invokes the read() = 2 operation after it observes the enqueue(2) operation (and after it invokes dequeue() = 1), but before it observes enqueue(3).

Lastly, we now explain why the following operations cannot be mapped into a valid sequential execution.
Suppose that there exists a valid sequential execution *e*.
We note that the following order must be respected in *e*:
- *dequeue() = 1 -> read() = {2}*: Because *enqueue(1) -> enqueue(2)* and *read() = {2}*, we conclude that *dequeue() = 1 -> read() = {2}*.
- *enqueue(3) -> read() = {1, 2, 3}*: Because *read() = {1, 2, 3}*, we conclude that this order must be satisfied.
- *read() = {1, 2, 3} -> dequeue() = 1*: Because the read operation "contains" 1, we conclude that *read() = {1, 2, 3} -> dequeue(1)*.

Given the previously defined order, we conclude that *enqueue(3) -> read() = {1, 2, 3} -> dequeue(1) -> read() = {2}* in *e*. 
However, since *enqueue(3) -> read(2) = {2}* and there does not exist the *dequeue() = 3* operation, we conclude that the read operation invoked by the baby blockchain must "contain" 3.
Hence, *e* is not valid.

#### CCQ is causally consistent

In order to show that CCQ is causally consistent, we need to show that both blockchains execute operations that are causally related in the correct and same order.

In order to complete the proof, we need to incorporate the read operations.
Recall that there is no communication between two blockchain in case of read operations.
Hence, a blockchain does not really execute (nor observe) other blockchain's read operation.
However, we do need to point out a moment in which the "abstract execution" takes place in order to prove the causal consistency of our implementation.

A blockchain A executes its own read operation R at the moment returning from the operation.
However, the other blockchain B executes this operation in moments as we define now:
- Let R not be causally preceded by any write operation invoked by blockchain A.
In this case, B executes R at the same moment as A.
- Let R be preceded by a write operation W.
In this case, B executes R immediately after it observes W.
Note that if R is causally preceded by a read operation R' invoked by A which is also preceded by W, then R is executed by B immediately after R' is executed by B (note that R' is executed after W).

Finally, we conclude the proof.
Consider any two operations A and B such that A --> B.
We show that both blockchains execute A before executing B.
Let us investigate all cases:
- A and B are invoked by the same blockchain:
Trivially, the invoking blockchain does execute A before executing B.
We now show that the other blockchain executes A before B:
1) If both operations are write operations, then the fact that IBC channels are ordered ensures that the other blockchain executes A before B.
2) If A is a write operation and B is a read operation, we conclude that B is executed at the other blockchain after the observation of the "first preceding" write operation W.
Since W = A or A --> W, we conclude that A is indeed executed before B at the other blockchain.
3) If A is a read operation and B is a write operation, we conclude that either A is executed at the same moment as it was executed in the invoking blockchain (which ensures that A is executed before B) or A is executed immediately after the observation of the "first preceding" write operation W (which ensures that A is executed before B since W --> B).
4) If both operations are read operations, we conclude that it is impossible for B to be executed before A at the other blockchain.

- A is a write operation and the blockchain that invokes B has observed A before invoking B:
If A and B are invoked by the same blockchain, A is executed before B at both blockchains (see the previous case).
Hence, we consider the case where A is invoked by blockchain C and B is invoked by blockchain D.
Therefore, C executes A before B (since B is executed at the earliest at the same moment as D executes B, which is after executing A).
Moreover, D executes A before B since it observes A before invoking B.

- There exists an operation C such that A --> C and C --> B:
Since this case eventually reduces to the previous two, we conclude that both blockchains execute A before B, which concludes the proof.

## Generalization

In this subsection, we generalize the approach presented above to multiple blockchains.

### System model

Consider a set of N blockchains such that every two blockchains are connected via an ordered IBC channel.
Moreover, we assume that each blockchain can invoke all operations.

### Modification of the approach given above

Since now we have multiple blockchain, we introduce vector clocks in order to capture the causal precedence of write operations.
A vector clock is simply an array of N elements (one per each blockchain) and it is associated with each write operation of each blockchain.
For example, suppose that write operation A invoked by a blockchain B is associated with vector clock V.
Let V[C] = x.
This simply means that operation A is invoked **after** B had observed an x-th write operation issued by blockchain C.
Hence, A operation is causally preceded by first x write operations issued by C.

We compare vector clocks using the "<=" relation:
For any two vector clocks V, V', V <= V' if and only if, for every index i, V[i] <= V'[i].

Now we explain the modification in detail.
Each blockchain maintains a local current vector clock V (initialized to all 0s).
Once a blockchain issues an operation, it associates a vector clock W with the operation, such that W[self] is a sequence number of the write operation and W[i] = V[i], for every i != self.

Lastly, once a blockchain receives an IBC packet about a write operation, it does not process this operation until W' <= V, where W' represents the vector clock associated with the received operation.
Lastly, local vector clock is updated after each new operation is processed.
Namely, once operation O issued by blockchain B is processed, the vector clock V is updated in the following manner: V[B] = V[B] + 1 and V[i] remains unchanged, for every i != B.
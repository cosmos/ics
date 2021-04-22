# Technical Specification

## Introduction

This document presents a technical specification for the **Cross-Chain Validation** protocol.
The basic idea of the Cross-Chain Validation protocol is to allow validators that are already securing some existing blockchain (parent blockchain) to also secure a "new" blockchain (baby blockchain).
The stake bonded at the parent blockchain guarantees that a validator behaves correctly at the baby blockchain.
Otherwise, the validator is slashed on the parent blockchain.

Therefore, at a high level, we can imagine the Cross-Chain Validation protocol to be concerned with following entities:
  - Parent blockchain: This is a blockchain that "provides" validators. Namely, "provided" validators have some stake at the parent blockchain. Any misbehavior of a validator is slashed at the parent blockchain. Moreover, parent blockchain manipulates the validator set of a chain that "borrows" validators from it.
  - Baby blockchain: Baby blockchain is a blockchain that is being secured by the parent blockchain. In other words, validators that secure and operate the baby blockchain are bonded on the parent blockchain. Any misbehavior of a validator at the baby blockchain is punished at the parent blockchain (i.e., the validator is slashed at the parent blockchain).
  - Causally consistent shared FIFO queue: This queue is used for communication among two blockchains, as we explain in the rest of the document.

Note that the protocol we present is generalized to multiple baby blockchains.
In other words, a single parent blockchain might have multiple baby blockchains under its "jurisdiction".
Hence, the Cross-Chain Validation protocol is concerned with more than two entities (one parent blockchain and potentially multiple baby blockchains).

### Properties

This subsection is devoted to defining properties that the Cross-Chain Validation protocol ensures.
Recall that the parent blockchain has an ability to demand a change to the validator set of the baby chain.
Moreover, we want to ensure the stake of validators of the baby blockchain are "frozen" at the parent chain.

We present the interface of the protocol:
  - Request: <ChangeValidatorSet, babyChain, V> - request made by the parent blockchain to change the validator set of babyChain using the validator set updates V.
  We assume that each validator set update is unique (e.g., each validator set update V could simply have a unique identifier).
  - Indication: <Unbonded, babyChain, V> - indication to the parent blockchain that the validator set update V (i.e., its "effect") has unbonded on the baby blockchain.

Hence, we aim to achieve the following properties:
- *Liveness*: If the parent blockchain demands a validator set change V for the baby chain, then the validator set of the baby blockchain reflects this demand.
- *Validator set change safety*: Suppose that the validator set of the baby blockchain changes from *V* to *V'*. Then, there exists a sequence *seq = V1, V2, ..., Vn*, where *n > 0*, of change validator set demands issued by the parent blockchain such that *apply(V, seq) = V'*.
- *Validator set change liveness*: If there exist infinitely many <ChangeValidatorSet, babyChain, V'> issued after <ChangeValidatorSet, babyChain, V>, then <Unbonded, babyChain, V> is eventually triggered.

Let us explain the validator set change liveness property.
First, note that <Unbonded, babyChain, Vlast> is never triggered if Vlast represent the **final** validator set change demand issued by the parent blockchain (the reason is that the validator set of the baby blockchain will never change after Vlast).
Hence, it might be enough for V not to be the last demanded change.
Unfortunatelly, that is not the case since multiple validator set change demands might be committed in the same block as the last demand.
That is why we demand infinitely many demands to be issued after V.
Note that if B represents the number of transactions committed in each block, then it is sufficient to demand B validator set change demands to be issued in order for V to eventually unbond.

## High-level design of the Cross-Chain Validation protocol

In this section, we provide an intuition behind our protocol.

We use the causally consistent FIFO queue for "communication" among two blockchains (parent and baby).
Namely, the parent blockchain enqueues new validator set changes and the baby blockchain dequeues these validator set changes once they have unbonded on the baby blockchain (which represents a signal that the stake of those validator could be set free by the parent blockchain).

- Parent blockchain: Once the parent blockchain demands a change of validator set, it enqueues the validator set.

- Baby blockchain: Once the baby blockchain observes that there exist new validator set demands, the baby blockchain applies all these changes.
Moreover, whenever the validator set of the baby blockchain is modified, the "old" validator set should start unbonding.
That is the baby blockchain uses the **local** *UnbondingQueue* queue.
Namely, whenever the validator set of the baby blockchain is modified, we enqueue to *UnbondingQueue* the sequence number of the validator set change demand that was **last** applied in order to obtain the old (i.e., changed) validator set.
Once the unbonding period elapses for the validator set, the appropriate entry is dequeued from the shared queue.

## Data Structures

We devote this section to defining the data structures used to represent the states of both parent and baby blockchain, as well as the packets exchanged by the two blockchains.

### Application data

#### Parent blockchain

- Q: The causally consistent FIFO shared queue.

#### Baby blockchain

- Q: The causally consistent FIFO shared queue.
- *UnbondingQueue*: Keeps track of validators that are currently unbonding.
- validatorSetSeqNum: Number of validator set change demands processed; initialized to 0.
- dequeueSeqNum: Number of elements dequeued from Q; initialized to 0.
- lastObserved: Element of Q that was observed during the last Q.read() operation; initializated to nil.

## Transitions

In this section, we informally discuss the state transitions that occur in our protocol.
We observe state transitions that are driven by a user (i.e., the new staking module on the parent chain), driven by the relayer and driven by elapsed time.

  - User-driven state transitions: These transitions "start" the entire process of changing the validator set of the baby blockchain.
  We assume that the will to change the validator set of the baby blockchain babyChain is expressed by invoking <ChangeValidatorSet, babyChain, V>.

  - Time-driven state transitions: These transitions are activated since some time has elapsed. As we will present in the rest of the document, time-driven state transitions help us determine when the unbonding period, measured at the baby blockchain, has elapsed for a validator set.

In the rest of the document, we will discuss the aforementioned state transitions in more detail.

## Function Definitions

### Parent blockchain

This subsection will present the functions executed at the parent blockchain.

```golang
// expresses will to modify the validator set of the baby blockchain;
func changeValidatorSet(
  babyChainId: ChainId
  valSetUpdate: Validator[]
) {
  // enqueue to the queue shared with babyChainId
  Q[babyChainId].enqueue(valSetUpdate)
}
```

- Expected precondition
  - There exists a blockchain with *babyChainId* identifier
  - All validators from *valSetUpdate* are validators at the parent blockchain
- Expected postcondition
  - The valSetUpdate is enqueued to the queue shared with the blockchain with *babyChainId*
- Error condition
  - If the precondition is violated

TODO discuss and write a staking logic on endBlock
Important: By reading the shared queue Q, the parent blockchain is able to conclude which validators could get their stake back.

### Baby blockchain

```golang
// initialization function
func init() {
  UnbondingQueue = new local FIFO queue
  validatorSetSeqNum = 0
  dequeueSeqNum = 0
  lastObserved = nil
}
```


```golang
// function used in End-Block method
// returns all the elements enqueued since the last invocation of this method
func observeChangesQ() { 
  // get the content of the shared queue
  content = Q.read()

  // check whether the last observed element is in content
  if (!content.has(lastObserved)) {
    // update the lastObserved to the last element of content if content is not empty
    if (!content.empty()) lastObserved = content.last()

    // return the entire content
    return content
  } else {
    // set "old" lastObserved
    oldLastObserved = lastObserved

    // update the lastObserved to the last element of content
    lastObserved = content.last()

    // return the entire content of content starting from oldLastObserved (excluding oldLastObserved)
    return content.startFrom(oldLastObserved)
  }
}
```

- Expected precondition
  - None
- Expected postcondition
  - Returns the elements enqueued since the last Q.read() operation was invoked.
- Error condition
  - None

```golang
// End-Block method executed at the end of each block
func endBlock(block: Block) {
  // get time
  time = block.time

  // finish unbonding for mature validator sets
  while (!UnbondingQueue.isEmpty()) {
    // peak the first queue entry
    seqNum, startTime = UnbondingQueue.peak()

    if (startTime + UNBONDING_PERIOD >= time) {
      // remove from the unbonding queue
      seqNum, startTime = UnbondingQueue.dequeue()

      // dequeue all elements until seqNum
      for (i = dequeueSeqNum + 1; i <= seqNum; i++) {
        Q.dequeue()
      }
      dequeueSeqNum = seqNum

    } else {
      break
    }
  }

  // get the new changes
  changes = observeChangesQ()

  // if there are no changes, return currnt validator set
  if (changes.empty()) {
    // validator set remains the same
    ABCI.updateValidatorSet(block.validatorSet)
    return
  }

  // get the old validator set
  oldValSet = block.validatorSet

  // get the new validator set; init to the old one
  newValSet = oldValSet

  // start unbonding for the old validator set represented by the validator set update;
  // "start unbonding" simply means adding to the queue of validator set changes that started unbonding
  UnbondingQueue.enqueue(validatorSetSeqNum, time)

  // update the validator set
  while (!changes.isEmpty()) {
    valSetUpdate = changes.dequeue()

    // update the new validator set
    newValSet = applyValidatorUpdate(newValSet, valSetUpdate)

    // increment validatorSetSeqNum
    validatorSetSeqNum++

    // remember which demands participate
    if (content.isEmpty()) {
      newSeqNum = validatorSetSeqNum
    }
  }

  return newValSet
}
```

- Expected precondition
  - Every transaction from the *block* is executed
- Expected postcondition
  - Unbonding starts for the old validator set
  - Unbonding finishes for all validator set that started unbonding more than *unbondingTime* before *time = block.time*. Moreover, the *UnbondingOver* packet is created for each such validator set
  - The new validator set *newValSet* is pushed to the Tendermint protocol and *newValSet* reflects all the change validator set demands from the *block*
- Error condition
  - If the precondition is violated


## Correctness Arguments

Here we provide correctness arguments for the liveness, validator set change safety and liveness properties.

### Liveness
Suppose that the IBC communication indeed successfully relays the change validator set demand to the baby blockchain.
Therefore, the validator set of the baby blockchain should reflect this demand.
This indeed happens at the end of a block, since every observed demand is applied in order for the baby blockchain to calculate the new validator set.
Hence, the property is satisfied.

### Validator set change safety
Suppose that the validator set of the baby blockchain changes from *V* to *V'* in two consecutive blocks.
By construction of the protocol, we conclude that there exists a sequence of change validator set demands issued by the parent blockchain that result in *V'* when applied to *V*.
Recursively, we conclude that this holds for any two validator sets (irrespectively of the "block distance" between them).

### Validator set change liveness

Since infinitely many demands are issued after the demand V, we conclude that eventually the unbonding period is started for V.
The unbonding period eventually elapses and the V is dequeued at that moment.
As soon as the dequeued is "observed" by the parent blockchain, the indication is triggered.
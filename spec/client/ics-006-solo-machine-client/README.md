---
ics: 6
title: Solo Machine Client
stage: draft
category: IBC/TAO
kind: instantiation
implements: 2
author: Christopher Goes <cwgoes@tendermint.com>
created: 2019-12-09
modified: 2019-12-09
---

## Synopsis

This specification document describes a client (verification algorithm) for a solo machine with a single updateable public key which implements the [ICS 2](../ics-002-client-semantics) interface.

### Motivation

Solo machines — which might be devices such as phones, browsers, or laptops — might like to interface with other machines & replicated ledgers which speak IBC, and they can do so through the uniform client interface.

Solo machine clients are roughly analogous to "implicit accounts" and can be used in lieu of "regular transactions" on a ledger, allowing all transactions to work through the unified interface of IBC.

### Definitions

Functions & terms are as defined in [ICS 2](../ics-002-client-semantics).

### Desired Properties

This specification must satisfy the client interface defined in [ICS 2](../ics-002-client-semantics).

Conceptually, we assume "big table of signatures in the universe" - that signatures produced are public - and incorporate replay protection accordingly.

## Technical Specification

This specification contains implementations for all of the functions defined by [ICS 2](../ics-002-client-semantics).

### Client state

The `ClientState` of a solo machine is simply whether or not the client is frozen.

```typescript
interface ClientState {
  frozen: boolean
  consensusState: ConsensusState
}
```

### Consensus state

The `ConsensusState` of a solo machine consists of the current public key, current diversifier, sequence number, and timestamp.

The diversifier is an arbitrary string, chosen when the client is created, designed to allow the same public key to be re-used across different solo machine clients (potentially on different chains) without being considered misbehaviour.

```typescript
interface ConsensusState {
  sequence: uint64
  publicKey: PublicKey
  diversifier: string
  timestamp: uint64
}
```

### Height

The `Height` of a solo machine is just a `uint64`, with the usual comparison operations.

### Headers

`Header`s must only be provided by a solo machine when the machine wishes to update the public key or diversifier.

```typescript
interface Header {
  sequence: uint64
  timestamp: uint64
  signature: Signature
  newPublicKey: PublicKey
  newDiversifier: string
}
```

### Misbehaviour 

`Misbehaviour` for solo machines consists of a sequence and two signatures over different messages at that sequence.

```typescript
interface SignatureAndData {
  sig: Signature
  data: []byte
}

interface Misbehaviour {
  sequence: uint64
  signatureOne: SignatureAndData
  signatureTwo: SignatureAndData
}
```

### Signatures

Signatures are provided in the `Proof` field of client state verification functions. They include data & a timestamp, which must also be signed over.

```typescript
interface Signature {
  data: []byte
  timestamp: uint64
}
```

### Client initialisation

The solo machine client `initialise` function starts an unfrozen client with the initial consensus state.

```typescript
function initialise(consensusState: ConsensusState): ClientState {
  return {
    frozen: false,
    consensusState
  }
}
```

The solo machine client `latestClientHeight` function returns the latest sequence.

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.consensusState.sequence
}
```

### Validity predicate

The solo machine client `checkValidityAndUpdateState` function checks that the currently registered public key has signed over the new public key with the correct sequence.

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header) {
  assert(header.sequence === clientState.consensusState.sequence)
  assert(header.timestamp >= clientstate.consensusState.timestamp)
  assert(checkSignature(header.newPublicKey, header.sequence, header.diversifier, header.signature))
  clientState.consensusState.publicKey = header.newPublicKey
  clientState.consensusState.diversifier = header.newDiversifier
  clientState.consensusState.timestamp = header.timestamp
  clientState.consensusState.sequence++
}
```

### Misbehaviour predicate

Any duplicate signature on different messages by the current public key freezes a solo machine client.

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  misbehaviour: Misbehaviour) {
    h1 = misbehaviour.h1
    h2 = misbehaviour.h2
    pubkey = clientState.consensusState.publicKey
    diversifier = clientState.consensusState.diversifier
    timestamp = clientState.consensusState.timestamp
    // assert that timestamp could have fooled the light client
    assert(misbehaviour.h1.signature.timestamp >= timestamp)
    assert(misbehaviour.h2.signature.timestamp >= timestamp)
    // assert that signature data is different
    assert(misbehaviour.h1.signature.data !== misbehaviour.h2.signature.data)
    // assert that the signatures validate
    assert(checkSignature(pubkey, misbehaviour.sequence, diversifier, misbehaviour.h1.signature.data))
    assert(checkSignature(pubkey, misbehaviour.sequence, diversifier, misbehaviour.h2.signature.data))
    // freeze the client
    clientState.frozen = true
}
```

### State verification functions

All solo machine client state verification functions simply check a signature, which must be provided by the solo machine.

Note that value concatenation should be implemented in a state-machine-specific escaped fashion.

```typescript
function verifyClientState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  counterpartyClientState: ClientState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/clientState")
    // ICS 003 will not increment the proof height after connection verification
    // the solo machine client must increment the proof height to ensure it matches 
    // the expected sequence used in the signature
    abortTransactionUnless(height + 1 == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + counterpartyClientState
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/consensusState/{consensusStateHeight}")
    // ICS 003 will not increment the proof height after connection or client state verification
    // the solo machine client must increment the proof height by 2 to ensure it matches 
    // the expected sequence used in the signature
    abortTransactionUnless(height + 2 == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + consensusState
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connection/{connectionIdentifier}")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + connectionEnd
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyChannelState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + channelEnd
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyPacketData(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/packets/{sequence}")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + data
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyPacketAcknowledgement(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + acknowledgement
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyPacketReceiptAbsence(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/receipts/{sequence}")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}

function verifyNextSequenceRecv(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/nextSequenceRecv")
    abortTransactionUnless(height == clientState.consensusState.sequence)
    abortTransactionUnless(!clientState.frozen)
    abortTransactionUnless(proof.timestamp >= clientState.consensusState.timestamp)
    value = clientState.consensusState.sequence + clientState.consensusState.diversifier + proof.timestamp + path + nextSequenceRecv
    assert(checkSignature(clientState.consensusState.pubKey, value, proof.sig))
    clientState.consensusState.sequence++
    clientState.consensusState.timestamp = proof.timestamp
}
```

### Properties & Invariants

Instantiates the interface defined in [ICS 2](../ics-002-client-semantics).

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Not applicable. Alterations to the client verification algorithm will require a new client standard.

## Example Implementation

None yet.

## Other Implementations

None at present.

## History

December 9th, 2019 - Initial version
December 17th, 2019 - Final first draft

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).

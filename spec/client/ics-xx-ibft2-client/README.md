---
ics: XX
title: IBFT 2.0 Client
stage: draft
category: IBC/TAO
kind: instantiation
author: Jun Kimura <jun.kimura@datachain.jp>
created: 2021-04-14
implements: 2
---

## Synopsis

This specification document describes a client (verification algorithm) for a blockchain using IBFT 2.0 consensus.

### Motivation

State machines of various sorts replicated using the IBFT 2.0 consensus protocol might like to interface with other replicated state machines or solo machines over IBC.

### Definitions

Functions & terms are as defined in [ICS 2](https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics).

### Desired Properties

This specification must satisfy the client interface defined in ICS 2.

## Technical Specification

This specification depends on correct instantiation of the [IBFT 2.0 consensus protocol](https://arxiv.org/abs/1909.10194) and [light client protocol](https://github.com/datachainlab/ibc-solidity/blob/main/docs/ibft2-light-client.md).

### Client State

The IBFT 2.0 client state tracks the trusting period, latest height, ibcStoreAddress, and a possible frozen height. The ibcStoreAddress represents the address of the contract account which stores the commitments.

```typescript
interface ClientState {
  chainId: uint64
  trustLevel: Fraction
  trustingPeriod: uint64
  ibcStoreAddress: Address
  latestHeight: uint64
  frozenHeight: Maybe<uint64>
}
```

### Consensus State

The IBFT 2.0 client tracks the timestamp (block time), validator set, and commitment root for all previously verified consensus states (these can be pruned after the trusting period has passed, but should not be pruned beforehand). The commitmentRoot is a storage root of the account corresponding to the ibcStoreAddress in the ClientState.

```typescript
interface ConsensusState {
  timestamp: uint64
  commitmentRoot: []byte
  validatorSet: List<Address>
}
```

## Headers

The IBFT 2.0 client headers include the ethHeader which represents the header in hypereldger besu, the trustedHeight, the accountProof, and the commit seals by the validators who committed the block. The accountProof represents an inclusion-proof in the state root of an account with a storage root corresponding to the commitment root. The header also implements the height, stateRoot, time, validatorSet functions with the ethHeader.

```typescript
interface Header {
  ethHeader: ETHHeader
  commitSeals: List<[]byte>
  trustedHeight: uint64
  accountProof: []byte

  height() => uint64
  stateRoot() => []byte
  timestamp() => uint64
  validatorSet() => List<Address>
}

interface ETHHeader {
  parentHash: Buffer
  sha3Uncles: Buffer
  miner: Address
  stateRoot: Buffer
  transactionsRoot: Buffer
  receiptsRoot: Buffer
  logsBloom: Buffer
  difficulty: bigint
  number: number
  gasLimit: bigint
  gasUsed: bigint
  timestamp: number
  extraData: Buffer
  mixHash: Buffer
  nonce: number
}
```

## Misbehaviour

The Misbehaviour type is used for detecting misbehaviour and freezing the client - to prevent further packet flow - if applicable. IBFT 2.0 client Misbehaviour consists of two headers at the same height both of which the light client would have considered valid.

```typescript
interface Misbehaviour {
  fromHeight: Height
  h1: Header
  h2: Header
}
```

## Client initialisation

IBFT 2.0 client initialisation requires a (subjectively chosen) latest consensus state, including the full validator set.

```typescript
function initialise(
  chainID: uint64, consensusState: ConsensusState,
  validatorSet: List<Address>, trustLevel: Fraction,
  height: Height, trustingPeriod: uint64): ClientState {
    assert(height > 0)
    assert(trustLevel > 0 && trustLevel < 1)
    set("clients/{identifier}/consensusStates/{height}", consensusState)
    return ClientState{
      chainID,
      validatorSet,
      trustLevel,
      latestHeight: height,
      trustingPeriod,
      frozenHeight: null
    }
}
```

The IBFT 2.0 client `latestClientHeight` function returns the latest stored height, which is updated every time a new (more recent) header is validated.

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.latestHeight
}
```

### Validity predicate

IBFT 2.0 client validity checking uses the verification logic described in [the IBFT 2.0 Light Client spec](https://github.com/datachainlab/ibc-solidity/blob/main/docs/ibft2-light-client.md). If the provided header is valid, the client state is updated & the newly verified commitment written to the store.

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  consensusState: ConsensusState,
  header: Header) {
    // assert trusting period has not yet passed
    assert(clientState.trustingPeriod == 0 || currentTimestamp() - clientState.latestTimestamp < clientState.trustingPeriod)
    // assert header timestamp is less than trust period in the future. This should be resolved with an intermediate header.
    assert(clientState.trustingPeriod == 0 || header.timestamp() - clientState.latestTimeStamp < clientState.trustingPeriod)
    // assert header timestamp is past current timestamp
    assert(header.timestamp() > clientState.latestTimestamp)
    // assert header height is newer than any we know
    assert(header.height() > clientState.latestHeight)
    // call the `verify` function
    assert(verify(consensusState.validatorSet, clientState.latestHeight, clientState.trustingPeriod, clientState.trustlevel, header))

    // update latest height
    clientState.latestHeight = header.height()

    // verify the inclusion of the account in the state root and get the account
    account = header.stateRoot().verifyMembershipAndGetLeaf(keccak256(abi.encodePacked(clientState.ibcStoreAddress)), header.accountProof);
    // ethereum account is a 4 item array of [nonce,balance,storageRoot,codeHash]
    commitmentRoot = account[2];

    // create recorded consensus state, save it
    consensusState = ConsensusState{timestamp: header.timestamp(), commitmentRoot: commitmentRoot, validatorSet: header.validatorSet()}
    set("clients/{identifier}/consensusStates/{header.height}", consensusState)
    // save the client
    set("clients/{identifier}", clientState)
}
```

### Misbehaviour predicate

IBFT 2.0 client misbehaviour checking determines whether or not two conflicting headers at the same height would have convinced the light client.

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  misbehaviour: Misbehaviour) {
    // assert that the heights are the same
    assert(misbehaviour.h1.height() === misbehaviour.h2.height())
    // assert that the commitments are different
    assert(misbehaviour.h1.commitmentRoot() !== misbehaviour.h2.commitmentRoot())
    // fetch the previously verified commitment root & validator set
    consensusState = get("clients/{identifier}/consensusStates/{misbehaviour.fromHeight}")
    // assert that the timestamp is not from more than an trusting period ago
    assert(clientState.trustingPeriod == 0 || currentTimestamp() - misbehaviour.h1.timestamp() < clientState.trustingPeriod && currentTimestamp() - misbehaviour.h2.timestamp() < clientState.trustingPeriod)
    // check if the light client "would have been fooled"
    assert(
      verify(consensusState.validatorSet, misbehaviour.fromHeight, misbehaviour.h1) &&
      verify(consensusState.validatorSet, misbehaviour.fromHeight, misbehaviour.h2)
      )
    // set the frozen height
    clientState.frozenHeight = min(clientState.frozenHeight, misbehaviour.h1.height) // which is same as h2.height
    // save the client
    set("clients/{identifier}", clientState)
}
```

### State verification functions

IBFT 2.0 client state verification functions check a Merkle proof against a previously validated commitment root.

The Merkle proof is based on [Merkle Patricia Trie in Ethereum](https://eth.wiki/en/fundamentals/patricia-tree#main-specification-merkle-patricia-trie).

```typescript
function verifyClientConsensusState(
  clientState: ClientState,
  height: Height,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: Height,
  consensusState: ConsensusState) {
    // calculate the storage slot of the consensus state
    path = consensusStateCommitmentSlot(prefix, clientIdentifier, consensusStateHeight)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided consensus state has been stored
    assert(root.verifyMembership(path, consensusState, proof))
}

function verifyConnectionState(
  clientState: ClientState,
  height: Height,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    // calculate the storage slot of the connection state
    path = connectionStateCommitmentSlot(prefix, connectionIdentifier)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided connection end has been stored
    assert(root.verifyMembership(path, connectionEnd, proof))
}

function verifyChannelState(
  clientState: ClientState,
  height: Height,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    // calculate the storage slot of the channel state
    path = channelStateCommitmentSlot(prefix, portIdentifier, channelIdentifier)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided channel end has been stored
    assert(root.verifyMembership(path, channelEnd, proof))
}

function verifyPacketData(
  clientState: ClientState,
  height: Height,
  delayPeriodTime: uint64,
  delayPeriodBlocks: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    // calculate the storage slot of the packet commitment
    path = packetCommitmentSlot(prefix, portIdentifier, channelIdentifier, sequence)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the processed time
    processedTime = get("clients/{identifier}/processedTimes/{height}")
    // fetch the processed height
    processedHeight = get("clients/{identifier}/processedHeights/{height}")
    // assert that enough time has elapsed
    assert(currentTimestamp() >= processedTime + delayPeriodTime)
    // assert that enough blocks have elapsed
    assert(currentHeight() >= processedHeight + delayPeriodBlocks)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    assert(root.verifyMembership(path, hash(data), proof))
}

function verifyPacketAcknowledgement(
  clientState: ClientState,
  height: Height,
  delayPeriodTime: uint64,
  delayPeriodBlocks: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    // calculate the storage slot of the packet acknowledgement commitment
    path = packetAcknowledgementCommitmentSlot(prefix, portIdentifier, channelIdentifier, sequence)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the processed time
    processedTime = get("clients/{identifier}/processedTimes/{height}")
    // fetch the processed height
    processedHeight = get("clients/{identifier}/processedHeights/{height}")
    // assert that enough time has elapsed
    assert(currentTimestamp() >= processedTime + delayPeriodTime)
    // assert that enough blocks have elapsed
    assert(currentHeight() >= processedHeight + delayPeriodBlocks)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided acknowledgement has been stored
    assert(root.verifyMembership(path, hash(acknowledgement), proof))
}

function verifyPacketReceiptAbsence(
  clientState: ClientState,
  height: Height,
  delayPeriodTime: uint64,
  delayPeriodBlocks: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    // calculate the storage slot of the packet receipt commitment
    path = packetReceiptCommitmentSlot(prefix, portIdentifier, channelIdentifier, sequence)
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the processed time
    processedTime = get("clients/{identifier}/processedTimes/{height}")
    // fetch the processed height
    processedHeight = get("clients/{identifier}/processedHeights/{height}")
    // assert that enough time has elapsed
    assert(currentTimestamp() >= processedTime + delayPeriodTime)
    // assert that enough blocks have elapsed
    assert(currentHeight() >= processedHeight + delayPeriodBlocks)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that no receipt has been stored
    assert(root.verifyNonMembership(path, proof))
}

function verifyNextSequenceRecv(
  clientState: ClientState,
  height: Height,
  delayPeriodTime: uint64,
  delayPeriodBlocks: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    // calculate the storage slot of the nextSequenceRecv
    path = packetNextSequenceRecvCommitmentSlot(prefix, portIdentifier, channelIdentifier)
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the processed time
    processedTime = get("clients/{identifier}/processedTimes/{height}")
    // fetch the processed height
    processedHeight = get("clients/{identifier}/processedHeights/{height}")
    // assert that enough time has elapsed
    assert(currentTimestamp() >= processedTime + delayPeriodTime)
    // assert that enough blocks have elapsed
    assert(currentHeight() >= processedHeight + delayPeriodBlocks)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the nextSequenceRecv is as claimed
    assert(root.verifyMembership(path, nextSequenceRecv, proof))
}
```

### Storage slot calculators for commitments

The layout of state variables in storage follows [the spec of solidity](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html).

```typescript

// This value is implementation-dependent.
const commitmentSlot = 0;

// Commitments represents the type of state variable in the storage where commitments are stored
type Commitments = Map<bytes32, bytes32>


function clientStateCommitmentSlot(prefix: CommitmentPrefix, clientIdentifier: string) :bytes32 {
consensusStateHeight: uint64): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/clients/{clientIdentifier}", commitmentSlot));
}

function consensusStateCommitmentSlot(prefix: CommitmentPrefix, clientIdentifier: string, consensusStateHeight: uint64): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/clients/{clientIdentifier}/consensusState/{consensusStateHeight}", commitmentSlot));
}

function connectionStateCommitmentSlot(prefix: CommitmentPrefix, connectionIdentifier: string): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/connections/{connectionIdentifier}", commitmentSlot));
}

function channelStateCommitmentSlot(prefix: CommitmentPrefix, portIdentifier: string, channelIdentifier: string): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/ports/{portIdentifier}/channels/{channelIdentifier}", commitmentSlot));
}

function packetCommitmentSlot(prefix: CommitmentPrefix, portIdentifier: string, channelIdentifier: string, sequence: uint64): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/ports/{portIdentifier}/channels/{channelIdentifier}/packets/{sequence}"), commitmentSlot));
}

function packetAcknowledgementCommitmentSlot(prefix: CommitmentPrefix, portIdentifier: string, channelIdentifier: string, sequence: uint64): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}", commitmentSlot));
}

function packetReceiptCommitmentSlot(prefix: CommitmentPrefix, portIdentifier: string, channelIdentifier: string, sequence: uint64): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/ports/{portIdentifier}/channels/{channelIdentifier}/receipts/{sequence}"), commitmentSlot));
}

function packetNextSequenceRecvCommitmentSlot(prefix: CommitmentPrefix, portIdentifier: string, channelIdentifier: string): bytes32 {
  return keccak256(abi.encodePacked("{prefix}/ports/{portIdentifier}/channels/{channelIdentifier}/nextSequenceRecv"), commitmentSlot));
}
```

### Properties & Invariants

Correctness guarantees as provided by [the IBFT 2.0 light client protocol](https://github.com/datachainlab/ibc-solidity/blob/main/docs/ibft2-light-client.md).

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Not applicable. Alterations to the client verification algorithm will require a new client standard.

## Example Implementation

- https://github.com/datachainlab/ibc-solidity/blob/main/contracts/core/IBFT2Client.sol

## Other Implementations

None at present.

## History

April 14th, 2021 - Initial version

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).

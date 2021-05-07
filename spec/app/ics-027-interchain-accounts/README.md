---
ics: 27
title: Interchain Accounts
stage: Draft
category: IBC/TAO
requires: 25, 26
kind: instantiation
author: Tony Yun <tony@chainapsis.com>, Dogemos <josh@tendermint.com>
created: 2019-08-01
modified: 2020-07-14
---

## Synopsis

This standard document specifies packet data structure, state machine handling logic, and encoding details for the account management system over an IBC channel between separate chains.

### Motivation

On Ethereum, there are two types of accounts: externally owned accounts, controlled by private keys, and contract accounts, controlled by their contract code ([ref](https://github.com/ethereum/wiki/wiki/White-Paper)). Similar to Ethereum's CA (contract accounts), interchain accounts are managed by another chain while retaining all the capabilities of a normal account (i.e. stake, send, vote, etc). While an Ethereum CA's contract logic is performed within Ethereum's EVM, interchain accounts are managed by another chain via IBC in a way such that the owner of the account retains full control over how it behaves.

### Definitions

**TODO: Make this a table and add more useful definitions for used keywords throughout the spec**

The IBC handler interface & IBC relayer module interface are as defined in [ICS 25](../ics-025-handler-interface) and [ICS 26](../ics-026-routing-module), respectively.

### Desired Properties

- Permissionless
- Fault containment: An interchain account must follow rules of its host chain, even in times of Byzantine behavior by the counterparty chain (the chain that manages the account)
- The chain that controls the account must process the results asynchronously and according to the chain's logic. The acknowledgment message should contain a result of an error as described in [ics-4](https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics#acknowledgement-envelope).
- Sending and receiving transactions will be processed in an ordered channel where packets are delivered exactly in the order in which they were sent. 
- Each ordered channel must only allow one interchain account to use it, otherwise, malicious users may be able to block transactions from being received. Practically, N accounts per N channels can be achieved by creating a new port for each user (owner of the interchain account) and creating a unique channel for each new registration request. Future versions of ics-27 will use partially ordered channels to allow multiple interchain accounts on the same channel to preserve account sequence ordering without being reliant on one another

## Technical Specification

The implementation of interchain accounts is non-symmetric. This means that each sending chain can have a different way to generate an interchain account and each receiving chain can have a different set of transactions that may be deserialised in different ways. For example, chains that use the Cosmos SDK will deserialise tx bytes using Protobuf, but if the counterparty chain is a smart contract on Ethereum, it may deserialise tx bytes by an ABI that is a minimal serialisation algorithm for the smart contract.

A chain can implement one or both parts to the interchain accounts protocol (sending and receiving). A sending chain registering and controlling an account does not necessarily have to allow other chains to register accounts on its own chain, and vice versa. 

The interchain account specification defines the general way to register an interchain account and transfer tx bytes. The counterparty chain is responsible for deserialising and executing the tx bytes, and the sending chain should know how the counterparty chain will handle the tx bytes in advance. Each chain-specific implementation should clearly document how serialization/deserialization of transactions happens and ensure the required packet data format is clearly outlined. 

Each chain must satisfy the following features to create an interchain account:

- New interchain accounts must not conflict with existing ones
- Each chain must keep track of which counterparty chain created each new interchain account

The chain must reject the transaction and must not make a state transition in the following cases:

- The IBC transaction fails to be deserialised
- The authentication step on the receiving chain (where the interchain account is hosted) fails 

#### Known Issues
##### Ordered Channels 
In an ordered channel, the next packet cannot be relayed until the previous packet has been relayed. If the previous packet is timed out, the ordered channel will be closed and the next packet can never be relayed. If multiple distrusting accounts are allowed to use a single ordered channel this creates an attack vector whereby malicious users can block packets from being received in a timely manner. An example of this may be a malicious user sending hundreds of packets that are never paid for. Other users trying to use this channel will have to wait for all of these packets to be fully relayed or receive valid proof of these packets having timed out before their own packets can be relayed. Therefore, multiplexing N interchain accounts on 1 ordered channel is not viable. This specification assumes the following: N interchain accounts will exist on N ordered channels. 

##### Unordered Channels 
An unordered channel lets packets be relayed in any order. A user can send 100 packets and never relay them. Another user submits one packet and can immediately relay just that one without having to relay all other packets. In theory interchain accounts can be implemented using an unordered channel so long as messages that are dependent on one another are packed into a single packet. However, this pattern differs from other standards such as how Tendermint's mempool operates. Rather than deviate from well defined standards for message processing it is recommended to not to use this approach or at a minimum clearly document how messages are processed on the receiving chain. 

##### Partially Ordered Channels 
Interchain accounts are a perfect use case for partially ordered channels, whereby the order is based on account sequences. However, this has yet to be specified and implemented in IBC. As of the time of writing this specification the current progress can be tracked [here](https://github.com/cosmos/ibc/issues/550). 

### Architecture Diagram
![](https://i.imgur.com/HX1h2u2.png)



### Authentication & Authorization
The sending chain (the chain registering and controlling an account) will implement its own authentication and authorization, which will determine who can create an interchain account and what type of transactions the registered accounts can invoke. One example of this may be a cosmos SDK chain that only allows the creation of an interchain account on behalf of the chains distribution module (community pool), whereby actions this interchain account takes are determined by governance proposals voted on by the token holders of the sending chain. Another example may be a smart contract that registers an interchain account and has a specific set of actions it is authorized to take.  

The receiving chain must implement authentication with regard to ensuring that the incoming messages are sent by the owner of the targeted interchain account. With regards to authorization, it is up to each chain-specific implementation to decide if the hosted interchain accounts have the authority to invoke all of the chain's message types or only a subset. This configuration may be set up at module initialization.

### Data Structures

```typescript
interface InterchainAccountModule {
  // Sending/Controlling side
  tryRegisterInterchainAccount(data: Uint8Array)
  tryRunTx(chainType: Uint8Array, data: any)
  // Recieving side
  createAccount(): Address
  deserialiseTx(txBytes: Uint8Array): Tx
  authenticateTx(tx: Tx): boolean
  runTx(tx: Tx): Result
}
```

**A chain can implement the entire interface, or decide to implement only the sending or receiving parts of the protocol.**


#### Sending Interface
The `tryRegisterInterchainAccount` method in the `InterchainAccountModule` interface defines the way to request the creation of an interchain account on a remote chain. The remote chain creates an interchain account using its own account creation logic. Due to the limitation of ordered channels, the recommended way to achieve this when calling `tryRegisterInterchainAccount` is to dynamically bind a new port with the port id set as the address of the owner of the account (if the port is not already bound), invoke `OpenChanInit` via an IBC module which will initiate the handshake process and emit an event signaling to a relayer to generate a new channel between both chains. The remote chain can then create the interchain account in the `ChanOpenTry` callback as part of the channel creation handshake process defined in [ics-4](https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics). Once the `chanOpenAck` callback fires the handshake-originating (sending) chain can assume the account registration is successful.  


The `tryRunTx` method in the`InterchainAccountModule` interface defines the way to create an outgoing packet for a specific chain type. The chain type determines how the IBC account transaction should be constructed and serialised for the receiving chain. The sending side should know in advance how the receiving side expects the incoming IBC packet to be structured. 

#### Recieving Interface

`createAccount` defines the way to determine the account's address by using the port & channel id. A newly created interchain account must not conflict with an existing account. Therefore, the host chain (the chain that the account will live on) must keep track of which blockchains have created an interchain account in order to verify the transaction signing authority in `authenticateTx`. `createAccount` should be called in the `chanOpenTry` callback as part of the channel creation handshake process defined in [ics-4](https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics).

`authenticateTx` validates the transaction and checks that the signers in the transaction have the right permissions (are the owners of the account in question).`

`runTx` executes a transaction after the transaction has been successfully authenticated.

### Packet Data
`InterchainAccountPacketData` contains an array of messages that an interchain account can run and a memo string that is sent to the receiving chain. The example below is defined as a proto encoded message but each chain can encode this differently. This should be clearly defined in the documentation for each chain specific implementation.

```typescript
message InterchainAccountPacketData  {
    repeated google.protobuf.Any messages = 1;
    string memo = 2;
}
```

The acknowledgement packet structure is defined as in [ics4](https://github.com/cosmos/cosmos-sdk/blob/v0.42.4/proto/ibc/core/channel/v1/channel.proto#L134-L147). The acknowledgement result should contain the chain-id and the address of the targetted interchain account. If an error occurs on the receiving chain the acknowledgement should contain the error message.

```typescript
message Acknowledgement {
  // response contains either a result or an error and must be non-empty
  oneof response {
    bytes  result = 21;
    string error  = 22;
  }
}
```

The ```InterchainAccountHook``` interface allows the source chain to receive results of executing transactions on an interchain account.
```typescript
interface InterchainAccountHook {
  onTxSucceeded(sourcePort:string, sourceChannel:string, txBytes: Uint8Array)
  onTxFailed(sourcePort:string, sourceChannel:string, txBytes: Uint8Array)
}
```

### Port & channel setup
Receiving chains (chains that will host interchain accounts) must always bind to a port with the id `interchain_account`. Sending chains will bind to ports dynamically, with each port id being the address of the interchain account owner.

The example below assumes a module is implementing the entire `InterchainAccountModule` interface. The `setup` function must be called exactly once when the module is created (perhaps when the blockchain itself is initialized) to bind to the appropriate port.

```typescript
function setup() {
  capability = routingModule.bindPort("interchain_account", ModuleCallbacks{
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

Once the `setup` function has been called, channels can be created via the IBC routing module.

### Channel lifecycle management

An interchain account module will accept new channels from any module on another machine, if and only if:

- The channel being created is ordered
- The version string is "ics27-1"

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only ordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // version not used at present
  abortTransactionUnless(version === "ics27-1")
}
```

```typescript
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === ORDERED)
  // assert that version is "ics27-1"
  abortTransactionUnless(version === "ics27-1")
  abortTransactionUnless(counterpartyVersion === "ics27-1")
  // create an interchain account 
  createAccount(counterpartyPortIdentifier, counterpartyChannelIdentifier)
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // port has already been validated
  // assert that version is "ics27-1"
  abortTransactionUnless(version === "ics27-1")
  // state change to keep track of successfully registered interchain account
  confirmInterchainAccountRegistration()
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

### Packet relay
`onRecvPacket` is called by the routing module when a packet addressed to this module has been received.

```typescript
function onRecvPacket(packet: Packet) {
  InterchainAccountPacketData data = packet.data
    const tx = deserialiseTx(packet.data.txBytes)
    abortTransactionUnless(authenticateTx(tx))
    try {
      const result = runTx(tx)

      return Acknowledgement{
        result: result
      }
    } catch (e) {
      // Return ack with error.
      return Acknowledgement{
        error: e.message,
    }
}
```

`onAcknowledgePacket` is called by the routing module when a packet sent by this module has been acknowledged.

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
    if (acknowledgement.result) {
      onTxSucceeded(packet.sourcePort, packet.sourceChannel, packet.data.data)
    } else {
      onTxFailed(packet.sourcePort, packet.sourceChannel, packet.data.data)
    }
    return
}
```


**TODO: how to handle timeouts?**
```typescript
function onTimeoutPacket(packet: Packet) {

}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // nothing is necessary
}
```

## Example Implementation

Repository for Cosmos-SDK implementation of ICS27: https://github.com/cosmos/interchain-accounts

## History

Aug 1, 2019 - Concept discussed

Sep 24, 2019 - Draft suggested

Nov 8, 2019 - Major revisions

Dec 2, 2019 - Minor revisions (Add more specific description & Add interchain account on Ethereum)

July 14, 2020 - Major revisions

April 27, 2021 - Redesign of ics27 specification

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).

---
ics: '4'
title: 通道和数据包语义
stage: draft
category: IBC/TAO
kind: instantiation
requires: 2, 3, 5, 24
author: Christopher Goes <cwgoes@tendermint.com>
created: '2019-03-07'
modified: '2019-08-25'
---

## Synopsis

The "channel" abstraction provides message delivery semantics to the interblockchain communication protocol, in three categories: ordering, exactly-once delivery, and module permissioning. A channel serves as a conduit for packets passing between a module on one chain and a module on another, ensuring that packets are executed only once, delivered in the order in which they were sent (if necessary), and delivered only to the corresponding module owning the other end of the channel on the destination chain. Each channel is associated with a particular connection, and a connection may have any number of associated channels, allowing the use of common identifiers and amortising the cost of header verification across all the channels utilising a connection & light client.

Channels are payload-agnostic. The modules which send and receive IBC packets decide how to construct packet data and how to act upon the incoming packet data, and must utilise their own application logic to determine which state transactions to apply according to what data the packet contains.

### Motivation

链间通信协议使用跨链消息传递模型。 外部中继进程将 IBC * 数据包* 从一条链中继到另一条链。链 `A` 和链 `B` 独立地确认新的块，并且从一个链到另一个链的数据包可能会被任意延迟、审查或重新排序。数据包对于中继器是可见的，并且可以通过任何中继进程从链中读取，然后提交给任何其他链。

> IBC 协议必须保证顺序（对于有序通道）和有且仅有一次投递，以允许应用程序推理两条链上已连接模块的组合状态。例如，一个应用程序可能希望允许单个通证化的资产在多个区块链之间转移并保留在多个区块链上，同时保留可替代性和供应量。当特定的IBC数据包提交到链` B `时，应用程序可以在链` B `上铸造资产凭据，并要求链` A `将等额的资产托管在链` A `上，直到以后凭相反的 IBC 数据包将凭证兑换回链` A `为止。这种顺序保证了正确的应用逻辑，可以确保两个链上的资产总量不变，并且在链` B `上铸造的所有资产凭证都可以在日后通过赎回回到链` A `上。

为了向应用层提供所需的排序、有且只有一次发送和模块许可语义，区块链间通信协议必须实现一种抽象以强制执行这些语义——通道就是这种抽象。

### Definitions

`ConsensusState` 在 [ICS 2](../ics-002-client-semantics) 中被定义.

`Connection` 在 [ICS 3](../ics-003-connection-semantics) 中被定义.

`Port`和`authenticate`在[ICS 5](../ics-005-port-allocation)中被定义。

`hash` is a generic collision-resistant hash function, the specifics of which must be agreed on by the modules utilising the channel.

`Identifier` ， `get` ， `set` ， `delete` ， `getCurrentHeight`和模块系统相关的原语在[ICS 24](../ics-024-host-requirements)中被定义。

*通道*是用于在单独的区块链上的特定模块之间进行有且仅有一次数据包传递的管道，该模块至少具备数据包发送端和数据包接收端。

A *bidirectional* channel is a channel where packets can flow in both directions: from `A` to `B` and from `B` to `A`.

*单向*通道是指数据包只能沿一个方向流动的通道：从`A`到`B` （或从`B`到`A` ，任意命名顺序）。

An *ordered* channel is a channel where packets are delivered exactly in the order which they were sent.

An *unordered* channel is a channel where packets can be delivered in any order, which may differ from the order in which they were sent.

```typescript
enum ChannelOrder {
  ORDERED,
  UNORDERED,
}
```

Directionality and ordering are independent, so one can speak of a bidirectional unordered channel, a unidirectional ordered channel, etc.

所有通道均提供有且仅有一次的数据包传送，这意味着在通道的一端发送的数据包最终将不多于且不少于一次地传送到另一端。

This specification only concerns itself with *bidirectional* channels. *Unidirectional* channels can use almost exactly the same protocol and will be outlined in a future ICS.

通道的末端是一条链上存储通道元数据的数据结构：

```typescript
interface ChannelEnd {
  state: ChannelState
  ordering: ChannelOrder
  counterpartyPortIdentifier: Identifier
  counterpartyChannelIdentifier: Identifier
  connectionHops: [Identifier]
  version: string
}
```

- The `state` is the current state of the channel end.
- The `ordering` field indicates whether the channel is ordered or unordered.
- `counterpartyPortIdentifier`标识通道另一端的对应链上的端口号。
- `counterpartyChannelIdentifier`标识对应链的通道端。
- `nextSequenceSend`是单独存储的，用于追踪下一个将要发送的数据包的序列号。
- `nextSequenceRecv`是单独存储的，追踪要接收的下一个数据包的序列号。
- `connectionHops`按顺序存储连接标识符列表，在此通道上发送的数据包将沿该标识符行进。目前，此列表的长度必须为1。将来可能会支持多跳通道。
- `version`字符串存储一个不透明的通道版本号，该版本号在握手期间已达成共识。这可以确定模块级别的配置，例如通道使用哪种数据包编码。核心 IBC 协议不会使用该版本号。

通道端具有以下*状态* ：

```typescript
enum ChannelState {
  INIT,
  OPENTRY,
  OPEN,
  CLOSED,
}
```

- 处于`INIT`状态的通道端，表示刚刚开始了握手的建立。
- 处于`OPENTRY`状态的通道端，表示已在对应链上确认了握手这一步。
- 处于`OPEN`状态的通道端，表示已完成握手，并为发送和接收数据包作好了准备。
- 处于`CLOSED`状态的通道端，表示通道已关闭，不能再用于发送或接收数据包。

链间通信协议中的`Packet`是如下定义的特定接口：

```typescript
interface Packet {
  sequence: uint64
  timeoutHeight: uint64
  sourcePort: Identifier
  sourceChannel: Identifier
  destPort: Identifier
  destChannel: Identifier
  data: bytes
}
```

- `sequence`对应于发送和接收的顺序，其中数据包的发送和接受必须按照对应的序列号顺序来进行。
- `timeoutHeight`标示目标链上不再处理数据包之后的共识高度，用于记录已经超时的区块高度。
- `sourcePort`标识发送链上的端口号。
- `sourceChannel`标识发送链上的通道端。
- `destPort`标识接收链上的端口号。
- `destChannel`标识接收链上的通道端。
- The `data` is an opaque value which can be defined by the application logic of the associated modules.

请注意， `Packet`永远不会直接序列化。而是在某些函数调用中使用的中间结构，可能需要由调用 IBC 处理程序的模块来创建或处理该中间结构。

`OpaquePacket`是一个数据包，但是被状态机主机掩盖为一种模糊的数据类型，因此，除了将其传递给 IBC 处理程序之外，模块无法对其进行任何操作。 IBC 处理程序可以将`Packet`转换为`OpaquePacket` ，反之亦然。

```typescript
type OpaquePacket = object
```

### 设计的属性

#### 效能

- 数据包传输和确认的速度应仅受底层链速度的限制。证明应尽可能是批量化的。

#### 有且仅有一次传递

- 在通道的一端发送的 IBC 数据包应准确地传递到另一端。
- 对于有且仅有一次安全性，不需要网络同步假设。如果其中一条链或两条链都停止了，则数据包最多只能传递一次，并且一旦链恢复，数据包就应该能够再次流动。

#### 按序

- On ordered channels, packets should be sent and received in the same order: if packet *x* is sent before packet *y* by a channel end on chain `A`, packet *x* must be received before packet *y* by the corresponding channel end on chain `B`.
- 在无序通道上，可以以任何顺序发送和接收数据包。像有序数据包一样，无序数据包的超时是分别地对应目标链上的特定区块高度发生的。

#### Permissioning

- 通道应该在握手期间被通道的两端都允许，并且此后不可变更（更高级别的逻辑可以通过标记端口的所有权来标记通道所有权）。只有与通道端关联的模块才能在其上发送或接收数据包。

## Technical Specification

### Dataflow visualisation

客户端、连接、通道和数据包的体系结构：

![Dataflow Visualisation](../../../../spec/ics-004-channel-and-packet-semantics/dataflow.png)

### Preliminaries

#### Store paths

通道的结构存储在结合端口标识符和通道标识符的唯一的存储路径前缀下：

```typescript
function channelPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "ports/{portIdentifier}/channels/{channelIdentifier}"
}
```

The capability key associated with a channel is stored under the `channelCapabilityPath`:

```typescript
function channelCapabilityPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
  return "{channelPath(portIdentifier, channelIdentifier)}/key"
}
```

`nextSequenceSend`和`nextSequenceRecv`无符号整数计数器分开进行存储，因此可以单独证明它们：

```typescript
function nextSequenceSendPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/nextSequenceSend"
}

function nextSequenceRecvPath(portIdentifier: Identifier, channelIdentifier: Identifier): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/nextSequenceRecv"
}
```

Constant-size commitments to packet data fields are stored under the packet sequence number:

```typescript
function packetCommitmentPath(portIdentifier: Identifier, channelIdentifier: Identifier, sequence: uint64): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/packets/" + sequence
}
```

0 位等同于存储中的路径的缺失。

Packet acknowledgement data are stored under the `packetAcknowledgementPath`:

```typescript
function packetAcknowledgementPath(portIdentifier: Identifier, channelIdentifier: Identifier, sequence: uint64): Path {
    return "{channelPath(portIdentifier, channelIdentifier)}/acknowledgements/" + sequence
}
```

无序通道必须始终向该路径写入确认信息（甚至是空的），使得此类确认的缺失可以用作超时证明。有序通道可以写一个确认信息，但不是必须的。

### Versioning

在握手过程中，通道的两端在与该通道关联的版本字节串上达成一致。 此版本字节串的内容对于 IBC 核心协议不透明。
状态机主机可以利用版本数据来标示其支持的 IBC / APP 协议，认同数据包编码格式，或在 IBC 协议之上协商与自定义逻辑有关的其他通道元数据。

状态机主机可以安全地忽略版本数据或指定一个空字符串。

### Sub-protocols

> 注意：如果主机状态机正在使用对象能力认证（请参阅[ICS 005](../ics-005-port-allocation) ），则所有使用端口的功能都将带有附加能力参数。

#### Identifier validation

通道存储在唯一的`(portIdentifier, channelIdentifier)`前缀下。
或将提供验证函数`validatePortIdentifier` 。

```typescript
type validateChannelIdentifier = (portIdentifier: Identifier, channelIdentifier: Identifier) => boolean
```

If not provided, the default `validateChannelIdentifier` function will always return `true`.

#### 通道生命周期管理

![Channel State Machine](../../../../spec/ics-004-channel-and-packet-semantics/channel-state-machine.png)

发起者 | 数据报 | 作用链 | 先前状态 (A, B) | 作用后状态（A，B）
--- | --- | --- | --- | ---
参与者 | ChanOpenInit | A | (none, none) | (INIT, none)
Relayer | ChanOpenTry | B | (INIT, none) | (INIT, TRYOPEN)
Relayer | ChanOpenAck | A | (INIT, TRYOPEN) | (OPEN, TRYOPEN)
Relayer | ChanOpenConfirm | B | (OPEN, TRYOPEN) | (OPEN, OPEN)

Initiator | Datagram | 作用链 | Prior state (A, B) | 作用后状态（A，B）
--- | --- | --- | --- | ---
参与者 | ChanCloseInit | A | (OPEN, OPEN) | (CLOSED, OPEN)
Relayer | ChanCloseConfirm | B | (CLOSED, OPEN) | (CLOSED, CLOSED)

##### 建立握手

模块调用`chanOpenInit`函数，以与另一个链上的模块初始化通道建立握手。

建立通道必须提供本地通道标识符、本地端口、远程端口和远程通道标识符的标识符。

当打开握手完成后，发起握手的模块将拥有在账本上已创建通道的一端，而对应的另一条链的模块将拥有通道的另一端。创建通道后，所有权就无法更改（尽管更高级别的抽象可以实现并提供此功能）。

```typescript
function chanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string): CapabilityKey {
    abortTransactionUnless(validateChannelIdentifier(portIdentifier, channelIdentifier))

    abortTransactionUnless(connectionHops.length === 1) // for v1 of the IBC protocol

    abortTransactionUnless(provableStore.get(channelPath(portIdentifier, channelIdentifier)) === null)
    connection = provableStore.get(connectionPath(connectionHops[0]))

    // optimistic channel handshakes are allowed
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state !== CLOSED)
    abortTransactionUnless(authenticate(privateStore.get(portPath(portIdentifier))))
    channel = ChannelEnd{INIT, order, counterpartyPortIdentifier,
                         counterpartyChannelIdentifier, connectionHops, version}
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
    key = generate()
    provableStore.set(channelCapabilityPath(portIdentifier, channelIdentifier), key)
    provableStore.set(nextSequenceSendPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceRecvPath(portIdentifier, channelIdentifier), 1)
    return key
}
```

模块调用`chanOpenInit`函数，以与另一个链上的模块初始化通道建立握手。

```typescript
function chanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string,
  proofInit: CommitmentProof,
  proofHeight: uint64): CapabilityKey {
    abortTransactionUnless(validateChannelIdentifier(portIdentifier, channelIdentifier))
    abortTransactionUnless(connectionHops.length === 1) // for v1 of the IBC protocol
    abortTransactionUnless(provableStore.get(channelPath(portIdentifier, channelIdentifier)) === null)
    abortTransactionUnless(authenticate(privateStore.get(portPath(portIdentifier))))
    connection = provableStore.get(connectionPath(connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{INIT, order, portIdentifier,
                          channelIdentifier, connectionHops.reverse(), counterpartyVersion}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofInit,
      counterpartyPortIdentifier,
      counterpartyChannelIdentifier,
      expected
    ))
    channel = ChannelEnd{OPENTRY, order, counterpartyPortIdentifier,
                         counterpartyChannelIdentifier, connectionHops, version}
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
    key = generate()
    provableStore.set(channelCapabilityPath(portIdentifier, channelIdentifier), key)
    provableStore.set(nextSequenceSendPath(portIdentifier, channelIdentifier), 1)
    provableStore.set(nextSequenceRecvPath(portIdentifier, channelIdentifier), 1)
    return key
}
```

发起握手的模块调用`chanOpenAck` ，以确认握手请求已
被对应另一条链上的模块所接受。

```typescript
function chanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyVersion: string,
  proofTry: CommitmentProof,
  proofHeight: uint64) {
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel.state === INIT)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(portIdentifier, channelIdentifier))))
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{OPENTRY, channel.order, portIdentifier,
                          channelIdentifier, channel.connectionHops.reverse(), counterpartyVersion}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofTry,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = OPEN
    channel.version = counterpartyVersion
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

握手接受模块调用`chanOpenConfirm`函数以确认在另一条链上握手发起模块的答复，并完成通道建立握手。

```typescript
function chanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  proofAck: CommitmentProof,
  proofHeight: uint64) {
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPENTRY)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(portIdentifier, channelIdentifier))))
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{OPEN, channel.order, portIdentifier,
                          channelIdentifier, channel.connectionHops.reverse(), channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofAck,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = OPEN
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

##### Closing handshake

两个模块中的任意一个通过调用`chanCloseInit`函数来关闭其通道端。一旦一端关闭，通道将无法重新打开。

在调用`chanCloseInit`的同时，可以选择调用其他模块去执行恰当的应用逻辑。

通道关闭后，任何传递中的数据包都可以是超时的。

```typescript
function chanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(portIdentifier, channelIdentifier))))
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    channel.state = CLOSED
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

一旦一端关闭了连接，另一端的模块调用`chanCloseConfirm`函数以关闭其通道端。
因为另一端已经关闭。

在调用`chanCloseConfirm`的同时，可以选择调用其他模块去执行恰当的应用逻辑。

一旦通道关闭，通道将无法重新打开。

```typescript
function chanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  proofInit: CommitmentProof,
  proofHeight: uint64) {
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(portIdentifier, channelIdentifier))))
    channel = provableStore.get(channelPath(portIdentifier, channelIdentifier))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    expected = ChannelEnd{CLOSED, channel.order, portIdentifier,
                          channelIdentifier, channel.connectionHops.reverse(), channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofInit,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))
    channel.state = CLOSED
    provableStore.set(channelPath(portIdentifier, channelIdentifier), channel)
}
```

#### Packet flow & handling

![Packet State Machine](../../../../spec/ics-004-channel-and-packet-semantics/packet-state-machine.png)

##### 数据包旅程

对于要从机器* A *上的模块* 1 *发送到机器* B *上的模块* 2 *的数据包，必须遵循以下步骤顺序，从头开始。

该模块可以通过 [ICS 25](../ics-025-handler-interface) 或 [ICS 26](../ics-026-routing-module) 接入 IBC 处理程序。

1. 以任何顺序初始客户端和端口设置
    1. 在 *A上* 为 *B* 创建客户端（请参阅 [ICS 2](../ics-002-client-semantics) ） 
    2. 在 *B上* 为 *A* 创建客户端（请参阅 [ICS 2](../ics-002-client-semantics) ） 
    3. 模块 *1* 绑定到端口（请参阅 [ICS 5](../ics-005-port-allocation) ） 
    4. 模块 *2* 绑定到端口（请参阅 [ICS 5](../ics-005-port-allocation) ），该端口通过不同信号端口与模块 *1* 通信
2. 建立连接和通道，乐观发送，以便
    1. 通过模块 *1* 从 *A* 到 *B* 开始进行连接打开握手（请参阅[ICS 3](../ics-003-connection-semantics) ） 
    2. 使用新创建的连接（此 ICS），通道打开握手从 *1* 开始到 *2* 
    3. 通过新创建的通道从 *1* 到 *2* 发送的数据包（此 ICS） 
3. Successful completion of handshakes (if either handshake fails, the connection/channel can be closed & the packet timed-out)
    1. Connection opening handshake completes successfully (see [ICS 3](../ics-003-connection-semantics)) (this will require participation of a relayer process)
    2. Channel opening handshake completes successfully (this ICS) (this will require participation of a relayer process)
4. Packet confirmation on machine *B*, module *2* (or packet timeout if the timeout height has passed) (this will require participation of a relayer process)
5. Acknowledgement (possibly) relayed back from module *2* on machine *B* to module *1* on machine *A*

Represented spatially, packet transit between two machines can be rendered as follows:

![Packet Transit](../../../../spec/ics-004-channel-and-packet-semantics/packet-transit.png)

##### Sending packets

The `sendPacket` function is called by a module in order to send an IBC packet on a channel end owned by the calling module to the corresponding module on the counterparty chain.

Calling modules MUST execute application logic atomically in conjunction with calling `sendPacket`.

The IBC handler performs the following steps in order:

- Checks that the channel & connection are open to send packets
- Checks that the calling module owns the sending port
- Checks that the packet metadata matches the channel & connection information
- Checks that the timeout height specified has not already passed on the destination chain
- Increments the send sequence counter associated with the channel
- Stores a constant-size commitment to the packet data & packet timeout

Note that the full packet is not stored in the state of the chain - merely a short hash-commitment to the data & timeout value. The packet data can be calculated from the transaction execution and possibly returned as log output which relayers can index.

```typescript
function sendPacket(packet: Packet) {
    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))

    // optimistic sends are permitted once the handshake has started
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state !== CLOSED)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.sourcePort, packet.sourceChannel))))
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)
    connection = provableStore.get(connectionPath(channel.connectionHops[0]))

    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state !== CLOSED)

    consensusState = provableStore.get(consensusStatePath(connection.clientIdentifier))
    abortTransactionUnless(consensusState.getHeight() < packet.timeoutHeight)

    nextSequenceSend = provableStore.get(nextSequenceSendPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(packet.sequence === nextSequenceSend)

    // all assertions passed, we can alter state

    nextSequenceSend = nextSequenceSend + 1
    provableStore.set(nextSequenceSendPath(packet.sourcePort, packet.sourceChannel), nextSequenceSend)
    provableStore.set(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence), hash(packet.data, packet.timeout))

    // log that a packet has been sent
    emitLogEntry("sendPacket", {sequence: packet.sequence, data: packet.data, timeout: packet.timeout})
}
```

#### Receiving packets

The `recvPacket` function is called by a module in order to receive & process an IBC packet sent on the corresponding channel end on the counterparty chain.

Calling modules MUST execute application logic atomically in conjunction with calling `recvPacket`, likely beforehand to calculate the acknowledgement value.

The IBC handler performs the following steps in order:

- Checks that the channel & connection are open to receive packets
- Checks that the calling module owns the receiving port
- Checks that the packet metadata matches the channel & connection information
- Checks that the packet sequence is the next sequence the channel end expects to receive (for ordered channels)
- Checks that the timeout height has not yet passed
- Checks the inclusion proof of packet data commitment in the outgoing chain's state
- Sets the opaque acknowledgement value at a store path unique to the packet (if the acknowledgement is non-empty or the channel is unordered)
- Increments the packet receive sequence associated with the channel end (ordered channels only)

```typescript
function recvPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  acknowledgement: bytes): Packet {

    channel = provableStore.get(channelPath(packet.destPort, packet.destChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.destPort, packet.destChannel))))
    abortTransactionUnless(packet.sourcePort === channel.counterpartyPortIdentifier)
    abortTransactionUnless(packet.sourceChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)

    abortTransactionUnless(getConsensusHeight() < packet.timeoutHeight)

    abortTransactionUnless(connection.verifyPacketCommitment(
      proofHeight,
      proof,
      packet.sourcePort,
      packet.sourceChannel,
      packet.sequence,
      hash(packet.data, packet.timeout)
    ))

    // all assertions passed (except sequence check), we can alter state

    if (acknowledgement.length > 0 || channel.order === UNORDERED)
      provableStore.set(
        packetAcknowledgementPath(packet.destPort, packet.destChannel, packet.sequence),
        hash(acknowledgement)
      )

    if (channel.order === ORDERED) {
      nextSequenceRecv = provableStore.get(nextSequenceRecvPath(packet.destPort, packet.destChannel))
      abortTransactionUnless(packet.sequence === nextSequenceRecv)
      nextSequenceRecv = nextSequenceRecv + 1
      provableStore.set(nextSequenceRecvPath(packet.destPort, packet.destChannel), nextSequenceRecv)
    }

    // log that a packet has been received & acknowledged
    emitLogEntry("recvPacket", {sequence: packet.sequence, timeout: packet.timeout, data: packet.data, acknowledgement})

    // return transparent packet
    return packet
}
```

#### Acknowledgements

The `acknowledgePacket` function is called by a module to process the acknowledgement of a packet previously sent by
the calling module on a channel to a counterparty module on the counterparty chain.
`acknowledgePacket` also cleans up the packet commitment, which is no longer necessary since the packet has been received and acted upon.

Calling modules MAY atomically execute appropriate application acknowledgement-handling logic in conjunction with calling `acknowledgePacket`.

```typescript
function acknowledgePacket(
  packet: OpaquePacket,
  acknowledgement: bytes,
  proof: CommitmentProof,
  proofHeight: uint64): Packet {

    // abort transaction unless that channel is open, calling module owns the associated port, and the packet fields match
    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.sourcePort, packet.sourceChannel))))
    abortTransactionUnless(packet.sourceChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    abortTransactionUnless(connection !== null)
    abortTransactionUnless(connection.state === OPEN)
    abortTransactionUnless(packet.sourcePort === channel.counterpartyPortIdentifier)

    // verify we sent the packet and haven't cleared it out yet
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeout))

    // abort transaction unless correct acknowledgement on counterparty chain
    abortTransactionUnless(connection.verifyPacketAcknowledgement(
      proofHeight,
      proof,
      packet.destPort,
      packet.destChannel,
      packet.sequence,
      hash(acknowledgement)
    ))

    // all assertions passed, we can alter state

    // delete our commitment so we can't "acknowledge" again
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

#### Timeouts

Application semantics may require some timeout: an upper limit to how long the chain will wait for a transaction to be processed before considering it an error. Since the two chains have different local clocks, this is an obvious attack vector for a double spend - an attacker may delay the relay of the receipt or wait to send the packet until right after the timeout - so applications cannot safely implement naive timeout logic themselves.

Note that in order to avoid any possible "double-spend" attacks, the timeout algorithm requires that the destination chain is running and reachable. One can prove nothing in a complete network partition, and must wait to connect; the timeout must be proven on the recipient chain, not simply the absence of a response on the sending chain.

##### Sending end

The `timeoutPacket` function is called by a module which originally attempted to send a packet to a counterparty module,
where the timeout height has passed on the counterparty chain without the packet being committed, to prove that the packet
can no longer be executed and to allow the calling module to safely perform appropriate state transitions.

Calling modules MAY atomically execute appropriate application timeout-handling logic in conjunction with calling `timeoutPacket`.

In the case of an ordered channel, `timeoutPacket` checks the `recvSequence` of the receiving channel end and closes the channel if a packet has timed out.

In the case of an unordered channel, `timeoutPacket` checks the absence of an acknowledgement (which will have been written if the packet was received). Unordered channels are expected to continue in the face of timed-out packets.

If relations are enforced between timeout heights of subsequent packets, safe bulk timeouts of all packets prior to a timed-out packet can be performed. This specification omits details for now.

```typescript
function timeoutPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecv: Maybe<uint64>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)

    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.sourcePort, packet.sourceChannel))))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // check that timeout height has passed on the other end
    abortTransactionUnless(proofHeight >= packet.timeoutHeight)

    // check that packet has not been received
    abortTransactionUnless(nextSequenceRecv < packet.sequence)

    // verify we actually sent this packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeout))

    if channel.order === ORDERED
      // ordered channel: check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecv
      ))
    else
      // unordered channel: verify absence of acknowledgement at packet index
      abortTransactionUnless(connection.verifyPacketAcknowledgementAbsence(
        proofHeight,
        proof,
        packet.sourcePort,
        packet.sourceChannel,
        packet.sequence
      ))

    // all assertions passed, we can alter state

    // delete our commitment
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    if channel.order === ORDERED {
      // ordered channel: close the channel
      channel.state = CLOSED
      provableStore.set(channelPath(packet.sourcePort, packet.sourceChannel), channel)
    }

    // return transparent packet
    return packet
}
```

##### Timing-out on close

模块调用` timeoutOnClose `函数，以证明未接收到数据包的寻址通道已关闭，因此将永远不会接收到该数据包（即使` timeoutHeight `尚未到达）。

```typescript
function timeoutOnClose(
  packet: Packet,
  proof: CommitmentProof,
  proofClosed: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecv: Maybe<uint64>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    // note: the channel may have been closed
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.sourcePort, packet.sourceChannel))))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // verify we actually sent this packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
           === hash(packet.data, packet.timeout))

    // check that the opposing channel end has closed
    expected = ChannelEnd{CLOSED, channel.order, channel.portIdentifier,
                          channel.channelIdentifier, channel.connectionHops.reverse(), channel.version}
    abortTransactionUnless(connection.verifyChannelState(
      proofHeight,
      proofClosed,
      channel.counterpartyPortIdentifier,
      channel.counterpartyChannelIdentifier,
      expected
    ))

    if channel.order === ORDERED
      // ordered channel: check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecv
      ))
    else
      // unordered channel: verify absence of acknowledgement at packet index
      abortTransactionUnless(connection.verifyPacketAcknowledgementAbsence(
        proofHeight,
        proof,
        packet.sourcePort,
        packet.sourceChannel,
        packet.sequence
      ))

    // all assertions passed, we can alter state

    // delete our commitment
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

##### 状态清理

模块调用`cleanupPacket`以从存储中删除收到的数据包承诺。接收端必须已经处理过该数据包（无论是定期清理还是仅针对过去的超时数据包的清理）。

In the ordered channel case, `cleanupPacket` cleans-up a packet on an ordered channel by proving that the packet has been received on the other end.

In the unordered channel case, `cleanupPacket` cleans-up a packet on an unordered channel by proving that the associated acknowledgement has been written.

```typescript
function cleanupPacket(
  packet: OpaquePacket,
  proof: CommitmentProof,
  proofHeight: uint64,
  nextSequenceRecvOrAcknowledgement: Either<uint64, bytes>): Packet {

    channel = provableStore.get(channelPath(packet.sourcePort, packet.sourceChannel))
    abortTransactionUnless(channel !== null)
    abortTransactionUnless(channel.state === OPEN)
    abortTransactionUnless(authenticate(privateStore.get(channelCapabilityPath(packet.sourcePort, packet.sourceChannel))))
    abortTransactionUnless(packet.destChannel === channel.counterpartyChannelIdentifier)

    connection = provableStore.get(connectionPath(channel.connectionHops[0]))
    // note: the connection may have been closed
    abortTransactionUnless(packet.destPort === channel.counterpartyPortIdentifier)

    // abortTransactionUnless packet has been received on the other end
    abortTransactionUnless(nextSequenceRecv > packet.sequence)

    // verify we actually sent the packet, check the store
    abortTransactionUnless(provableStore.get(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))
               === hash(packet.data, packet.timeout))

    if channel.order === ORDERED
      // check that the recv sequence is as claimed
      abortTransactionUnless(connection.verifyNextSequenceRecv(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        nextSequenceRecvOrAcknowledgement
      ))
    else
      // abort transaction unless acknowledgement on the other end
      abortTransactionUnless(connection.verifyPacketAcknowledgement(
        proofHeight,
        proof,
        packet.destPort,
        packet.destChannel,
        packet.sequence,
        nextSequenceRecvOrAcknowledgement
      ))

    // all assertions passed, we can alter state

    // clear the store
    provableStore.delete(packetCommitmentPath(packet.sourcePort, packet.sourceChannel, packet.sequence))

    // return transparent packet
    return packet
}
```

#### 竞态条件推导

##### 同时发生握手意图

If two machines simultaneously initiate channel opening handshakes with each other, attempting to use the same identifiers, both will fail and new identifiers must be used.

##### Identifier allocation

在目标链上分配标识符存在不可避免的竞争条件。最好建议模块使用伪随机、不表意的标识符。设法声明另一个模块希望使用的标识符，但是令人烦恼的是，由于不能在握手阶段陷入中间人攻击，因此接收模块必须已经拥有握手所指向的端口。

##### Timeouts / packet confirmation

There is no race condition between a packet timeout and packet confirmation, as the packet will either have passed the timeout height prior to receipt or not.

##### Man-in-the-middle attacks during handshakes

跨链状态的验证可防止连接握手和通道握手的中间人攻击，因为模块已知道所有信息（源、目标客户端、通道等），该信息将启动握手并在握手之前进行确认完成。

##### 有正在传输数据包时的连接/通道关闭

If a connection or channel is closed while packets are in-flight, the packets can no longer be received on the destination chain and can be timed-out on the source chain.

#### 通道查询

Channels can be queried with `queryChannel`:

```typescript
function queryChannel(connId: Identifier, chanId: Identifier): ChannelEnd | void {
    return provableStore.get(channelPath(connId, chanId))
}
```

### 属性和不变性

- 通道和端口标识符的唯一组合是“先到先得”的：分配了一对标识符组合后，只有拥有相应端口的模块才能在该通道上发送或接收。
- 假设链在超时后仍然存在，并且在发送链上出现了超时并有且仅有一次超时，数据报能够被有且只有一次地传送。
- 通道握手不能受到区块链上的另一个模块或另一个链的 IBC 处理程序作为中间人进行的攻击。

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

数据结构和编码可以在连接或通道级别进行版本控制。通道逻辑完全与数据包的数据格式无关，可以由模块在任何时候以自己喜欢的任何方式对其进行更改。

## 实现样例

Coming soon.

## 其他实现

Coming soon.

## History

Jun 5, 2019 - Draft submitted

Jul 4, 2019 - Modifications for unordered channels & acknowledgements

Jul 16, 2019 - Alterations for multi-hop routing future compatibility

Jul 29, 2019 - Revisions to handle timeouts after connection closure

Aug 13, 2019 - Various edits

Aug 25, 2019 - Cleanup

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).

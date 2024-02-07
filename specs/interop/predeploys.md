<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Predeploys

Two new system level predeploys are introduced for managing cross chain messaging along with
an update to the `L1Block` contract with additional functionality.

## CrossL2Inbox

| Constant | Value                                        |
|----------|----------------------------------------------|
| Address  | `0x4200000000000000000000000000000000000022` |

The `CrossL2Inbox` is responsible for executing a cross chain message on the destination chain.
It is permissionless to execute a cross chain message on behalf of any user. Certain protocol
enforced invariants must be preserved to ensure safety of the protocol.

### Invariants

- The timestamp of executing message MUST be greater than or equal to the timestamp of the initiating message
- The chain id of the initiating message MUST be in the dependency set
- The executing message MUST be initiated by an externally owned account such that the top level EVM
  call frame enters the `CrossL2Inbox`

#### Timestamp Invariant

The timestamp invariant ensures that initiating messages cannot come from a future block. Note that since
all transactions in a block have the same timestamp, it is possible for an executing transaction to be
ordered before the initiating message in the same block.

#### ChainID Invariant

Without a guarantee on the set of dependencies of a chain, it may be impossible for the derivation
pipeline to know which chain to source the initiating message from. This also allows for chain operators
to explicitly define the set of chains that they depend on.

#### Only EOA Invariant

The `onlyEOA` invariant on executing a cross chain message enables static analysis on executing messages.
This allows for the derivation pipeline and block builders to reject executing messages that do not
have a corresponding initiating message without needing to do any EVM execution.

It may be possible to relax this invariant in the future if the block building process is efficient
enough to do full simulations to gain the information required to verify the existence of the
initiating transaction. Instead of the `Identifier` being included in calldata, it would be emitted
in an event that can be used after the fact to verify the existence of the initiating message.
This adds complexity around mempool inclusion as it would require EVM execution and remote RPC
access to learn if a transaction can enter the mempool.

This feature could be added in a backwards compatible way by adding a new function to the `CrossL2Inbox`.

One possible way to handle explicit denial of service attacks is to utilize identity
in iterated games such that the block builder can ban identities that submit malicious transactions.

### Executing Messages

All of the information required to satisfy the invariants MUST be included in the calldata
of the function that is used to execute messages. Both the block builder and the smart contract use
this information to ensure that all system invariants are held.

The following fields are required for executing a cross chain message:

| Name      | Type         | Description                               |
|-----------|--------------|-------------------------------------------|
| `_target` | `address`    | Account that is called with `_msg`        |
| `_msg`    | `bytes`      | An opaque [Log][log]                      |
| `_id`     | `Identifier` | A pointer to the `_msg` in a remote chain |

[log]: https://github.com/ethereum/go-ethereum/blob/5c67066a050e3924e1c663317fd8051bc8d34f43/core/types/log.go#L29

#### `_target`

An `address` that is specified by the caller. There is no protocol enforcement on what this value is. The
`_target` is called with the `_msg`. In practice, the `_target` will be a contract that needs to know the
schema of the `_msg` so that it can be decoded. It MAY call back to the `CrossL2Inbox` to authenticate
properties about the `_msg` using the information in the `Identifier`.

#### `_msg`

Opaque `bytes` that represent a [Log][log]. The data is verified against the `Identifier`. It is serialized
by first concatenating the topics and then with the data.

```go
msg := make([]byte, 0)
for _, topic := range log.Topics {
    msg = append(msg, topic.Bytes()...)
}
msg = append(msg, log.Data...)
```

The `_msg` can easily be decoded into a Solidity `struct` with `abi.decode` since each topic is always 32 bytes and
the data generally ABI encoded.

#### `_id`

The `Identifier` that uniquely represents a log that is emitted from a chain. It can be considered to be a
unique pointer to a particular log. The derivation pipeline and fault proof program MUST ensure that the
`_msg` corresponds exactly to the log that the `Identifier` points to.

```solidity
struct Identifier {
    address origin;
    uint256 blocknumber;
    uint256 logIndex;
    uint256 timestamp;
    uint256 chainid;
}
```

| Name          | Type      | Description                                                                     |
|---------------|-----------|---------------------------------------------------------------------------------|
| `origin`      | `address` | Account that emits the log                                                      |
| `blocknumber` | `uint256` | Block number in which the log was emitted                                       |
| `logIndex`    | `uint256` | The index of the log in the array of all logs emitted in the block              |
| `timestamp`   | `uint256` | The timestamp that the log was emitted. Used to enforce the timestamp invariant |
| `chainid`     | `uint256` | The chainid of the chain that emitted the log                                   |

The `Identifier` includes the set of information to uniquely identify a log. When using an absolute
log index within a particular block, it makes ahead of time coordination more complex. Ideally there
is a better way to uniquely identify a log that does not add ordering constraints when building
a block. This would make building atomic cross chain messages more simple by not coupling the
exact state of the block templates between multiple chains together.

A simple implementation of the `executeMessage` function is included below.

```solidity
function executeMessage(address _target, bytes calldata _msg, Identifier calldata _id) public payable {
    require(msg.sender == tx.origin);
    require(_id.timestamp <= block.timestamp);
    require(L1Block.isInDependencySet(_id.chainid));

    assembly {
      tstore(ORIGIN_SLOT, _id.origin);
      tstore(BLOCKNUMBER_SLOT, _id.blocknumber)
      tstore(LOG_INDEX_SLOT, _id.logIndex)
      tstore(TIMESTAMP_SLOT, _id.timestamp)
      tstore(CHAINID_SLOT, _id.chainid)
    }

    bool success = SafeCall.call({
      _target: _target,
      _value: msg.value,
      _calldata: _msg
    });

    require(success);
}
```

By including the `Identifier` in the calldata, it makes static analysis much easier for block builders.
It is impossible to check that the `Identifier` matches the cross chain message on chain. If the block
builder includes a message that does not correspond to the `Identifier`, their block will be reorganized
by the derivation pipeline.

A possible upgrade path to this contract would involve adding a new function. If any fields in the `Identifier`
change, then a new 4byte selector will be generated by solc.

### `Identifier` Getters

The `Identifier` MUST be exposed via `public` getters so that contracts can call back to authenticate
properties about the `_msg`.

## L2ToL2CrossDomainMessenger

| Constant          | Value                                        |
|-------------------|----------------------------------------------|
| Address           | `0x4200000000000000000000000000000000000024` |
| `MESSAGE_VERSION` | `uint256(0)`                                 |
| `INITIAL_BALANCE` | `type(uint248).max`                          |

The `L2ToL2CrossDomainMessenger` is a higher level abstraction on top of the `CrossL2Inbox` that
provides features necessary for secure transfers of `ether` and ERC20 tokens between L2 chains.
Messages sent through the `L2ToL2CrossDomainMessenger` on the source chain receive both replay protection
as well as domain binding, ie the executing transaction can only be valid on a single chain.

### `relayMessage` Invariants

- Only callable by the `CrossL2Inbox`
- The `Identifier.origin` MUST be `address(L2ToL2CrossDomainMessenger)`
- The `_destination` chainid MUST be equal to the local chainid
- The `CrossL2Inbox` cannot call itself

### Message Versioning

Versioning is handled in the most significant bits of the nonce, similarly to how it is handled by
the `CrossDomainMessenger`.

```solidity
function messageNonce() public view returns (uint256) {
    return Encoding.encodeVersionedNonce(nonce, MESSAGE_VERSION);
}
```

### Transferring Ether in a Cross Chain Message

The `L2ToL2CrossDomainMessenger` MUST be initially set in state with an ether balance of `INITIAL_BALANCE`.
This initial balance exists to provide liquidity for cross chain transfers. It is large enough to always have
ether present to dispurse while still being able to accept inbound transfers of ether without overflowing.
The `L2ToL2CrossDomainMessenger` MUST only ensure all invariants are held before transferring out any ether.

The `L2CrossDomainMessenger` is not updated to include the `L2ToL2CrossDomainMessenger` functionality because
there is no need to introduce complexity between the L1 to L2 messages and the L2 to L2 liquidity.

### Interfaces

The `L2ToL2CrossDomainMessenger` uses a similar interface to the `L2CrossDomainMessenger` but
the `_minGasLimit` is removed to prevent complexity around EVM gas introspection and the `_destination`
chain is included instead.

#### Sending Messages

The initiating message is represented by the following event:

```solidity
event SentMessage(bytes message) anonymous;
```

The `bytes` are an ABI encoded call to `relayMessage`. The event is defined as `anonymous` so that no topics
are prefixed to the abi encoded call.

An explicit `_destination` chain and `nonce` are used to ensure that the message can only be played on a single remote
chain a single time. The `_destination` is enforced to not be the local chain to avoid edge cases.

```solidity
function sendMessage(uint256 _destination, address _target, bytes calldata _message) external payable {
    require(_destination != block.chainid);

    bytes memory data = abi.encodeCall(L2ToL2CrossDomainMessenger.relayMessage, (_destination, messageNonce(), msg.sender, _target, msg.value, _message));
    emit SentMessage(data);
    nonce++;
}
```

#### Relaying Messages

When relaying a message through the `L2ToL2CrossDomainMessenger`, it is important to require that
the `_destination` equal to `block.chainid` to ensure that the message is only valid on a single
chain. The hash of the message is used for replay protection.

It is important to ensure that the source chain is in the dependency set of the destination chain, otherwise
it is possible to send a message that is not playable.

```solidity
function relayMessage(uint256 _destination, uint256 _nonce, address _sender, address _target, uint256 _value, bytes memory _message) external {
    require(msg.sender == address(CROSS_L2_INBOX));
    require(_destination == block.chainid);
    require(CROSS_L2_INBOX.origin() == address(this));
    require(_target != address(this));

    bytes32 messageHash = keccak256(abi.encode(_destination, _nonce, _sender, _target, _value, _message));
    require(sentMessages[messageHash] == false);

    assembly {
      tstore(CROSS_DOMAIN_MESSAGE_SENDER_SLOT, _sender)
    }

    bool success = SafeCall.call({
       _target: _target,
       _value: _value,
       _calldata: _message
    });

    require(success);

    sentMessages[messageHash] = true;
}
```

## L1Block

| Constant | Value                                        |
|----------|----------------------------------------------|
| Address  | `0x4200000000000000000000000000000000000015` |

The `L1Block` contract is updated to include the set of allowed chains. The L1 Attributes transaction
sets the set of allowed chains. The `L1Block` contract MUST provide a public getter to check if a particular
chain is in the dependency set called `isInDependencySet(uint256)`. This function MUST return true when
the chain's chain id is passed in as an argument.

The `setL1BlockValuesInterop()` function MUST be called on every block after the interop upgrade block.
The interop upgrade block itself MUST include a call to `setL1BlockValuesEcotone`.


### L1Attributes

The L1 Atrributes transaction is updated to include the dependency set. Since the dependency set is dynamically sized,
a `uint8` "interopSetSize" parameter prefixes tightly packed `uint256` values that represent each chain id.

| Input arg         | Type                    | Calldata bytes          | Segment |
|-------------------|-------------------------|-------------------------|---------|
| {0x760ee04d}      | bytes4                  | 0-3                     | n/a     |
| baseFeeScalar     | uint32                  | 4-7                     | 1       |
| blobBaseFeeScalar | uint32                  | 8-11                    |         |
| sequenceNumber    | uint64                  | 12-19                   |         |
| l1BlockTimestamp  | uint64                  | 20-27                   |         |
| l1BlockNumber     | uint64                  | 28-35                   |         |
| basefee           | uint256                 | 36-67                   | 2       |
| blobBaseFee       | uint256                 | 68-99                   | 3       |
| l1BlockHash       | bytes32                 | 100-131                 | 4       |
| batcherHash       | bytes32                 | 132-163                 | 5       |
| interopSetSize    | uint8                   | 164-165                 | 6       |
| chainIds          | [interopSetSize]uint256 | 165-(32*interopSetSize) | 6+      |

## Security Considerations

TODO
# IBC

"IBC Protocol" refers to the Inter-Blockchain Communication Protocol. Also this document refers it to the IBC Protocol (IBC), which Cosmos is standardizing as Interchain Standards (ICS).

```
+---------------------------------------------------------------------------------------------+
| Distributed Ledger A                                                                        |
|                                                                                             |
| +----------+     +----------------------------------------------------------+               |
| |          |     | IBC Module                                               |               |
| | Module A | --> |                                                          | --> Consensus |
| |          |     | Handler --> Packet --> Channel --> Connection --> Client |               |
| +----------+     +----------------------------------------------------------+               |
+---------------------------------------------------------------------------------------------+

    +---------+
==> | Relayer | ==>
    +---------+

+--------------------------------------------------------------------------------------------+
| Distributed Ledger B                                                                       |
|                                                                                            |
|               +---------------------------------------------------------+     +----------+ |
|               | IBC Module                                              |     |          | |
| Consensus --> |                                                         | --> | Module B | |
|               | Client -> Connection --> Channel --> Packet --> Handler |     |          | |
|               +---------------------------------------------------------+     +----------+ |
+--------------------------------------------------------------------------------------------+
```
    https://github.com/cosmos/ics/blob/master/ibc/2_IBC_ARCHITECTURE.md#diagram


For more information about ICS in Cosmos, please refer to [repository on github](https://github.com/cosmos/ics).

In the following sections, we will specifically mention the specifications that are required for Fabric IBC to implement IBC.

## Client

Client refers to the light client defined in [ICS-002](https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics) on IBC.
The purpose of the Client is to allow a Blockchain communicating with an IBC to verify updates to the State agreed upon by the other Blockchain.

Updates to the Client are made by submitting a Header. After the submitted header is verified, the internally maintained ConsensusState and ClientState are updated.

### ConsensusState

The ConsensusState is used to validate the Header by Validity Predicate (see below).
https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics#consensusstate

For the definition in Fabric-IBC, please refer to [ConsensusState](05_fabric-client-spec.md#consensusstate).

### ClientState

ClientState is used to verify the Proof that a particular key/value pair exists or not in the State at a given Height.
https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics#clientstate


### Header

Header contains information to update the ConsensusState.
See https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics#header

See [Header](05_fabric-client-spec.md#header) for Fabric-IBC definition.

### Validity Predicate.

This refers to a function that validates the Header based on the current ConsensusState.
https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics#validity-predicate

In Fabric-IBC, when validating the Header, it also validates the Endorsed Commitment (described below).
The following verification functions are used internally.

#### VerifyChaincodeHeader
This verifies that the proposal response which contains ChaincodeHeader in the WriteSet is signed according to the currently registered endorsement policy.

#### VerifyChaincodeInfo
This verifies that ChaincodeInfo has been signed according to the currently registered IBC policy.

For details, see the following.
[Validity Predicate](05_fabric-client-spec.md#validity-predicate)

### State Verification Function

This refers to the function that verifies the internal state of the State that the Client should track.

In Fabric-IBC, when verifying the State, it also verifies the Endorsed Commitment as described below.
The following function is [defined](https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics#state-verification) by ICS.

#### verifyClientConsensusState
Verifies the Proof to the ConsensusState held in a specific Client on the target blockchain.

#### verifyConnectionState
Verify the Proof to a specific ConnectionState held on the target blockchain.

#### verifyChannelState
Verifies the Proof to a specific IBC ChannelState held on the target blockchain.

#### verifyPacketData
Verifies the Proof to a Packet originated at a specific IBC Channel, Port, or Sequence.

#### verifyPacketAcknowledgement
Verifies the Proof to a Packet Ack received at a specific IBC Channel, Port and Sequence.

#### verifyPacketAcknowledgementAbsence
Verifies the Proof of missing Packet Ack to be received at a specific IBC Channel, Port and Sequence.

#### verifyNextSequenceRecv
Verifies the Proof of the next Sequence to be received on a specific IBC Channel and Port.

For more information on Fabric-IBC, please refer to the following.
[State Verification Functions](05_fabric-client-spec.md#state-verification-functions)

### Endorsed Commitment

IBCs require Proof of Commitment to verify that a key/value pair exists (or does not exist) in the State of the target blockchain, with a small computational cost. This detail is defined in [ICS-023](https://github.com/cosmos/ics/tree/master/spec/ics-023-vector-commitments).

In Fabric-IBC, a Proof is created by querying a Chaincode, which returns a proposal response, to an Endorser that satisfies the Endorsement Policy maintained in the ClientState, such that the Read-Write Set contains the above pair. We define this as an Endorsed Commitment.

The verification of the Endorsed Commitment is to confirm that the signature on the proposal response satisfies the Endorsement Policy in the ClientState described above.

[Fabric IBC Modules](#fabric-ibc-modules) are provided for creating aforemetioned Endorsed Commitment.

For more information on Endorsed Commitment, please refer to the following
[Fabric Client Spec](05_fabric-client-spec.md)

## Connection, Channel

In IBC, Connection is defined in [ICS-03](https://github.com/cosmos/ics/tree/master/spec/ics-003-connection-semantics) and Channel is defined in [ICS-04](https://github.com/cosmos/ics/tree/master/spec/ics-004-channel-and-packet-semantics).

A Connection is a state that is held by each of the two blockchains communicating with the IBC, and is used in association with the Client. To communicate between Blockchains, a Connection must be established.

The Channel provides the semantics for Message delivery between each Module on the two blockchains using the Connection.
See the respective ICS for details.

In Fabric-IBC, the state transition models of Connection and Channel also follow IBC, while Endorsed Commitment is used for verification at each state transition.

Please refer to the following flow chart for Connection and Channel establishment.
[Connection, Channel establishment](04_architecture.md#connectionchannel establishment)

## Packet

Packet is defined in [ICS-04](https://github.com/cosmos/ics/tree/master/spec/ics-004-channel-and-packet-semantics).
In order for a Relayer to relay between blockchains, it has information about the IBC Channel, Port used by both the source and destination, and data. The data is defined by the logic implemented in each module.

## Relayer

Relayer is defined in [ICS-018](https://github.com/cosmos/ics/tree/master/spec/ics-018-relayer-algorithms).
It is an off-chain process defined by IBC that has the ability to read the state of transactions on a blockchain and relay them as Packets between blockchains.

As [mentioned](https://github.com/cosmos/ics/tree/master/spec/ics-018-relayer-algorithms#desired-properties) on ICS, a Relayer should satisfy the following properties In Fabric-IBC, these properties are also satisfied.

- The presence of a Relayer with Byzantine behavior does not compromise the important safety properties that IBC should satisfy, namely exactly-once and deliver-or-timeout.
- If there is at least one Relayer that is behaving normally, the liveness of the Packet relay is maintained.
- Relay itself can be done by anyone, and any verification required for Relay is done on the blockchain.


### Relayer Initialization

In Fabric-IBC, the Relayer is given a config related to the two IBC Chaincodes to be connected before the service is started.

#### Identity on Fabric ####
Relayer is a service that acts as a client of Fabric. Therefore, it is necessary to assign a Role that satisfies the following permissions using a CA.

1. can query events to the sender blockchain.
2. can query the State of the sender blockchain.
3. can submit Transaction to both blockchains.

Therefore, Relayer needs to be able to allocate the necessary Identity on both blockchains to satisfy 3. In addition, the local MSP information of both blockchains should be associated with the respective Chaincode.

#### Connection information for the peer of both Chaincodes ####
In order to fulfill the above authority, it is necessary to manage the connection information for each peer that operates the two Chaincodes.

#### Information on various IDs of IBCs ####
These values are [the same](https://github.com/datachainlab/cross/blob/adbf051333acb1f8fbcbae6ff2888d477a4bfeb9/tests/demo/path01.json#L1) as in Cosmos Relayer.
The clientID, connectionID, channelID, port, etc. of both Chaincodes should be set first.

### Relayer and Privacy

In Fabric-IBC, there is a potential privacy issue as relaying a State on a Permissioned blockchain to another blockchain. See below for a summary of this.

[On Relayer's privacy](06_appendix.md#relayer's privacy)

## Fabric-IBC Modules

Fabric-IBC Modules are a set of chaincodes that provide functions to generate Proof and header required by Fabric clients for state verification. The following two are provided.

- Chaincode header generator
- Endorsed commitment generator
### Chaincode header generator

The Chaincode header generator (CHG) is provided as a chaincode executed by the Endorser. It manages the current Sequence value and timestamps in the StateDB.
It provides the following functions;

- API for incrementing the sequence value
- Generates an Endorsed Commitment for the current or past sequence specified by the client.

### Endorsed commitment generator

The Endorsed commitment generator (ECG) generates an Endorsed Commitment as a Proof for reference to a specific commitment in the IBC's state.
It is provided as a chaincode to be executed by the Endorser. APIs are provided for each function for state verification.


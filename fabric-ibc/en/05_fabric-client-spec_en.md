# Fabric-client Spec

Fabric-client is a client for Hyperledger fabric that satisfies [ICS-02](https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics).

## ClientState

```typescript
interface ClientState {
  id: string
  chaincodeHeader: ChaincodeHeader
  ibcPolicy: Policy
  chaincodeInfo: ChaincodeInfo
  mspInfos: MSPInfos
}

interface ChaincodeHeader {
  sequence: commitment.Sequence // Value & Timestamp
}

interface ChaincodeInfo {
  channelId: string
  chaincodeId: ChaincodeID
  endorsementPolicy: []byte
}

interface Policy {
  policy: []byte
  sequence: uint64
}

interface MSPInfos {
 infos: []MSPInfo
}

interface MSPInfo {
  mspID: string // unique for its ClientState
  config: []byte // proto.Marshal(msppb.MSPConfig)
  policy: Policy
  frozen: bool // avoid reusing ID after deletion
}
```

ClientState tracks Endorsement policy, Chaincode ID and Version.

## ConsensusState

```typescript
interface ConsensusState {
  timestamp: int64
}
```

ConsensusState tracks Timestamp on IBC chaincode managed by Endorser.

## Header

Header includes either or both ChaincodeHeader and ChaincodeInfo.

```typescript
interface Header {
  chaincodeHeader: Maybe<ChaincodeHeader>
  ibcPolicy: Maybe<PolicyHeader>
  chaincodeInfo: Maybe<ChaincodeInfoHeader>
  mspInfoHeaders: Maybe<MSPInfoHeaders>
}

interface ChaincodeHeader {
  sequence: commitment.Sequence // contains Value, Timestamp
  proof: CommitmentProof
}

interface PolicyHeader {
  policy: Policy
  proof: MessageProof
}

interface ChaincodeInfoHeader {
  channelId: string
  chaincodeId: ChaincodeID
  endorsementPolicy: []byte
  ibcPolicySequence: uint64
  proof: MessageProof
}

interface MSPInfoHeaders {
  headers: []MSPInfoHeader
}

type MSPInfoHeaderType = "Create" | "UpdatePolicy" | "UpdateConfig" | "Freeze"

interface MSPInfoHeader {
  type: MspInfoHeaderType
  mspID: string
  config: []byte // proto.Marshal(msppb.MSPConfig)
  // when type is "UpdateConfig", policy.Sequence must be needed.
  policy: Maybe<Policy>
  ibcPolicySequence: uint64
  proof: MessageProof
}

```

### ChaincodeHeader.Proof

This acts as a proof of the result of the proposal by the fabric client, which is required to update the Sequene in the ConsensusState. The included signature must satisfy the Endorsement Policy held by the ClientState.

```typescript
interface CommitmentProof {
  proposal: []byte
  nsIndex: uint32
  writeSetIndex: uint32
  identities: [][]byte
  signatures: [][]byte
}
```

### PolicyHeader.Proof

This acts as a proof to the IBC Policy that is required to update the IBC Policy in the ClientState. The signature included must satisfy the IBC Policy held by the ClientState.

```typescript
interface MessageProof {
  identities: [][]byte
  signatures: [][]byte
}
```

### ChaincodeInfo.Proof

This act as a proof to the ChaincodeInfo that is required to update the ChaincodeInfo in the ClientState. The included signature must satisfy the IBC Policy held by the ClientState.

Similar to PolicyHeader.Proof, MessageProof is used.

## Misbehaviour

Misbehaviour in Fabric-IBC consists of two different Proofs for the same Sequence value for Sequence in the ClientState, or two Proofs for different values for the same key with regard to a Endorsed Commitment in the state verification.

```typescript
type Misbehaviour = MisbehaviourSubHeader | MisbehaviourEndorsedCommitment

type SubHeader = ChaincodeHeader | PolicyHeader | ChaincodeInfoHeader | MSPInfoHeader

type SequenceType = "ChaincodeHeader" | "IbcPolicy" | "MSPInfo"

type StateType =
  | "ClientConsensusState" | "ConnectionState" | "ChannelState"
  | "PacketCommitment" | "PacketAcknowledgement" | "PacketAcknowledgementAbsense"
  | "NextSequenceRecv"

interface MisbehaviourSubHeader {
  sequence: uint64
  sequenceType: SequenceType
  h1: SubHeader
  h2: SubHeader
}

interface MisbehaviourEndorsedCommitment {
  key: string
  stateType: StateType
  p1: CommitmentProof
  p2: CommitmentProof
}
```

### Update of Sequence values


It is necessary to make the order of sign deterministic, also it is necessary to make sure that each Signature used in Proof cannot be reused.

The Sequence value is initialized at the time of CreateClient, and thereafter is incremented when other state updates are performed with reference to the state associated with the Sequence value.

| update order | update target | Sequence value on ClientState | Notes | 
| ---- | ---------------- | ------------------------------------------------- | --------------------------------- |
| 0    | chaincodeHeader  | chaincodeHeader.sequence
| 1    | ibcPolicy.policy | ibcPolicy.sequence                            |                                   |
| 2    | chaincodeInfo    | ibcPolicy.sequence                            |                                   |
| 3 | mspInfoHeader.policy | ibcPolicy.sequence | The update order of each MSP in mspInfoHeaders is in ascending order of the MSPID. |
| 3 | mspInfoHeader.config | mspInfoHeaders[i].policy.sequence (i is the index of the corresponding MSP) | The update order of each MSP in mspInfoHeaders is in ascending order of the MSPID. |

When one Header contains multiple SubHeader updates, the Sequence value in Proof for each update must be incremented.

## Validity predicate

This function verifies that each signature is attached to the given Header according to the Endorsement policy that currently registered with.

```typescript
function checkValidity(clientState: ClientState, header: Header) {
  assert(header.chaincodeHeader != null || header.ibcPolicy != null || header.chaincodeInfo != null || header.MSPHeaders != null)
  if header.chaincodeHeader != null {
    assert(header.chaincodeHeader.sequence.timestamp > clientState.GetLatestTimestamp())
    h := clienttypes.NewHeight(0, header.chaincodeHeader.sequence.value)
    lh := clientState.GetLatestHeight()
    assert(h.EQ(clienttypes.NewHeight(lh.GetVersionNumber(), lh.GetVersionHeight()+1)))
    assert(verifyChaincodeHeader(clientState, header.chaincodeHeader)
  }

  if header.ibcPolicy != null {
    assert(verifyPolicyHeader(clientState, header.ibcPolicy))
  }

  if header.chaincodeInfo != null {
    assert(len(header.chaincodeInfo.endorsementPolicy) > 0)
    assert(verifyChaincodeInfo(clientState, header.chaincodeInfo))
  }

  if header.MSPHeaders != null {
    assert(verifyMSPHeaders(clientState, header.mspHeaders))
  }
}
```

The Header contains one or both of ChaincodeHeader and ChaincodeInfo elements. The following validation functions are used to verify each of these.


### VerifyChaincodeHeader

For the proposal response that contains ChaincodeHeader in the WriteSet, this function makes sure that it is signed according to the currently registered endorsement policy.

```typescript
function verifyChaincodeHeader(clientState: ClientState, h: ChaincodeHeader) {
  configs = getConfigs(clientState.mspInfos)
  lastci = clientState.chaincodeInfo
  assert(verifyEndorsedCommitment(clientState.chaincodeInfo.GetFabricChaincodeID(), lastci.endorsementPolicy, h.proof, MakeSequenceCommitmentEntryKey(h.sequence.value), h.sequence.Bytes(), configs))
}

// verifyEndorsedCommitment verifies a key-value entry with a policy
function verifyEndorsedCommitment(ccID: ChaincodeID, policyBytes: []byte, proof: CommitmentProof, key: string, value: []byte, configs: []MSPPBConfig) {
  policy = getPolicyEvaluator(policyBytes, configs)
  assert(policy != null)
  assert(policy.EvaluateSignedData(proof.ToSignedData()))

  id, rwset = getTxReadWriteSetFromProposalResponsePayload(proof.Proposal)
  assert(id != null || rwset != null)
  assert(equalChaincodeID(ccID, id))
  assert(ensureWriteSetIncludesCommitment(rwset.nsRwSets, proof.nsIndex, proof.writeSetIndex, key, value))
}
```

### VerifyPolicyHeader

For a ProposalResponse that contains a PolicyHeader in its WriteSet, this function confirms that it is signed according to the currently registered IBC policy.

```typescript
function verifyPolicyHeader(clientState: ClientState, ph: PolicyHeader) {
  assert(ph.policy.sequence == clientState.ibcPolicy.sequence + 1)
  configs = getConfigs(clientState.mspInfos)
  assert(verifyEndorsedMessage(clientState.ibcPolicy, ph.proof, ph.GetSignBytes(), configs))
}

// verifyEndorsedMessage verifies a value with given policy
function verifyEndorsedMessage(policyBytes: []byte, proof: MessageProof, value: []byte, configs: []msppb.MSPConfig) {
  policy = getPolicyEvaluator(policyBytes, configs)
  sigs = makeSignedDataListWithMessageProof(proof, value)
  assert(policy.EvaluateSignedData(sigs))
}
```

### VerifyChaincodeInfo

This function verifies that the ChaincodeInfo is signed according to the IBC policy currently registered.

```typescript
function verifyChaincodeInfo(clientState: ClientState, info: ChaincodeInfoHeader) {
  assert(cih.policy.sequence == clientState.ibcPolicy.sequence + 1)
  configs = getConfigs(clientState.mspInfos)
  assert(verifyEndorsedMessage(clientState.ibcPolicy, cih.proof, cih.GetSignBytes(), configs)
}
```

### VerifyMSPInfoHeaders

This function verifies that the required sign is presented according to the MSPInfo of each currently registered MSP for registering and updating Policy and Config.


```typescript
// assume reqs are sorted and each entry is unique
function verifyMSPInfoHeaders(clientState: ClientState, mihs: MSPInfoHeaders) {
  for _, mih := range mihs.headers {
  assert(verifyMSPInfoHeader(clientState, mih))
  }
}

function verifyMSPInfoHeader(clientstate: ClientState, mih: MSPInfoHeader) {
  switch mih.Type {
  case "Create":
    assert(verifyMSPCreate(clientState, mh))
    break
  case "UpdateConfig":
    assert(verifyMSPUpdateConfig(clientState, mh))
    break
  case "UpdatePolicy":
    assert(verifyMSPUpdatePolicy(clientState, mh))
    break
  case "Freeze":
    assert(verifyMSPFreeze(clientState, mh))
    break
  default:
    assert(false)
  }
}

function verifyMSPCreate(clientState: ClientState, mh: MSPHeader) {
  // error if already created

  assert(mh.ibcPolicySequence == clientState.ibcPolicy.sequence + 1)
  assert(mh.policy.sequence == 0)
  assert(verifyMSPWithPolicy(clientState.ibcPolicy, mh))
}

function verifyMSPUpdatePolicy(clientState: ClientState, mh: MSPHeader) {
  // error if not created yet

  assert(mh.ibcPolicySequence == clientState.ibcPolicy.sequence + 1)
  assert(verifyMSPWithPolicy(clientState.ibcPolicy, mh))
}

function verifyMSPUpdateConfig(clientState: ClientState, mh: MSPHeader) {
  // error if not created yet

  mi = findMSPInfo(clientState, mh.mspID)
  assert(mi != null)
  assert(mh.policy.sequence == mi.policy.sequence + 1)
  assert(VerifyMSPWithPolicy(mi.policy, mh))
}

function verifyMSPFreeze(clientState: ClientState, mh: MSPHeader) {
  // error if not created yet or already freezed

  assert(mh.ibcPolicySequence == clientState.ibcPolicy.sequence + 1)
  assert(VerifyMSPWithPolicy(clientState.ibcPolicy, mh))
}

function verifyMSPWithPolicy(policy: []byte, mh: MSPHeader) {
  config = clientState.mspInfos
  assert(verifyEndorsedMessage(policy, mh.Proof(), mh.GetSignBytes(), config))
}
```

## State verification functions

The various State verification functions perform the verification process by providing the Endorsed commitment (EC) generated by the Endorser as Proof.
The EC is obtained by the client of the fabric network by sending a transaction proposal to the Endorser that executes a chaincode that provides the Endorsed commitment generator (ECG).

### VerifyClientConsensusState

This verifies Proof to ConsensusState held in an individual Client on the target blockchain.

```go
// VerifyClientConsensusState verifies a proof of the consensus state of the specified client stored on the target machine.
func (cs ClientState) VerifyClientConsensusState(
    _ sdk.KVStore,
    cdc codec.Marshaler,
    aminoCdc *codec.Codec,
    _ commitmentexported.Root,
    height uint64,
    counterpartyClientIdentifier string,
    consensusHeight uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    bz, err := aminoCdc.MarshalBinaryBare(consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakeConsensusStateCommitmentEntryKey(prefix, counterpartyClientIdentifier, consensusHeight)
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyConnectionState

This verifies Proof to individual ConnectionState held on the target blockchain.

```go
// VerifyConnectionState verifies a proof of the connection state of the
// specified connection end stored locally.
func (cs ClientState) VerifyConnectionState(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    connectionID string,
    connectionEnd connectionexported.ConnectionI,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakeConnectionStateCommitmentEntryKey(prefix, connectionID)
    connection, ok := connectionEnd.(connectiontypes.ConnectionEnd)
    if !ok {
        return sdkerrors.Wrapf(sdkerrors.ErrInvalidType, "invalid connection type %T", connectionEnd)
    }
    bz, err := cdc.MarshalBinaryBare(&connection)
    if err != nil {
        return err
    }
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyChannelState

This verifies Proof to an individual ChannelState held on the target blockchain.

```go
// VerifyChannelState verifies a proof of the channel state of the specified
// channel end, under the specified port, stored on the target machine.
func (cs ClientState) VerifyChannelState(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    portID,
    channelID string,
    channel channelexported.ChannelI,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakeChannelStateCommitmentEntryKey(prefix, portID, channelID)
    channelEnd, ok := channel.(channeltypes.Channel)
    if !ok {
        return sdkerrors.Wrapf(sdkerrors.ErrInvalidType, "invalid channel type %T", channel)
    }
    bz, err := cdc.MarshalBinaryBare(&channelEnd)
    if err != nil {
        return err
    }
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketCommitment

This verifies the Proof to the Packet originated at a specific IBC Channel, Port, and Sequence.

```go
// VerifyPacketCommitment verifies a proof of an outgoing packet commitment at
// the specified port, specified channel, and specified sequence.
func (cs ClientState) VerifyPacketCommitment(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    portID,
    channelID string,
    sequence uint64,
    commitmentBytes []byte,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakePacketCommitmentEntryKey(prefix, portID, channelID, sequence)
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, commitmentBytes); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketAcknowledgement

This verifies the Proof to the Packet Ack received at a specific IBC Channel, Port and Sequence.

```go
// VerifyPacketAcknowledgement verifies a proof of an incoming packet
// acknowledgement at the specified port, specified channel, and specified sequence.
func (cs ClientState) VerifyPacketAcknowledgement(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    portID,
    channelID string,
    sequence uint64,
    acknowledgement []byte,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakePacketAcknowledgementEntryKey(prefix, portID, channelID, sequence)
    bz := channeltypes.CommitAcknowledgement(acknowledgement)
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketAcknowledgementAbsence

This verifies Proof of missing Packet Ack to be received at a specific IBC Channel, Port and Sequence.

```go
// VerifyPacketAcknowledgementAbsence verifies a proof of the absence of an
// incoming packet acknowledgement at the specified port, specified channel, and
// specified sequence.
func (cs ClientState) VerifyPacketAcknowledgementAbsence(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    portID,
    channelID string,
    sequence uint64,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakePacketAcknowledgementAbsenceEntryKey(prefix, portID, channelID, sequence)
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, []byte{}); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyNextSequenceRecv

This verifies Proof for the next Sequence to be received on a specific IBC Channel and Port.

```go
// VerifyNextSequenceRecv verifies a proof of the next sequence number to be
// received of the specified channel at the specified port.
func (cs ClientState) VerifyNextSequenceRecv(
    store sdk.KVStore,
    cdc codec.Marshaler,
    height uint64,
    prefix commitmentexported.Prefix,
    proof []byte,
    portID,
    channelID string,
    nextSequenceRecv uint64,
    consensusState clientexported.ConsensusState,
) error {
    fabProof, err := sanitizeVerificationArgs(cdc, cs, height, prefix, proof, consensusState)
    if err != nil {
        return err
    }
    key := commitment.MakeNextSequenceRecvEntryKey(prefix, portID, channelID)
    bz := sdk.Uint64ToBigEndian(nextSequenceRecv)
    if ok, err := VerifyEndorsement(cs.ChaincodeInfo.GetFabricChaincodeID(), cs.ChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

## Misbehaviour Predicate

This function confirms that the two given SubHeaders have a consensus failure for the current ClientState.

Whether or not to put the Client into a frozen state after Misbehaviour confirmation will be further discussed in the future based on the fact that Hyperledger Fabric is used for Enterprise.

```typescript
function checkMisbehaviourAndUpdateState(clientState: ClientState, misbehaviour: Misbehaviour) {
  switch (typeof misbehaviour) {
    case "MisbehaviourSubHeader":
      checkMisbehaviourSubHeaderAndUpdateState(clientState, misbehaviour)
      break
    case "MisbehaviourEndorsedCommitment":
      checkMisbehaviourEndorsedCommitmentAndUpdateState(clientState, misbehaviour)
      break
    default:
      assert(false)
      break
  }
}

function checkMisbehaviourSubHeaderAndUpdateState(clientState: ClientState, misbehaviour: MisbehaviourSubHeader) {
  switch (misbehaviour.sequenceType) {
    case "ChaincodeHeader":
      checkMisbehaviourForChaincodeHeaderSequence(clientState, misbehaviour)
      // clientState.frozen = true
      break
    case "IbcPolicy":
      checkMisbehaviourForPolicySequence(clientState, misbehaviour)
      // clientState.frozen = true
      break
    case "MSPInfo":
      checkMisbehaviourForMSPInfoHeaderSequence(clientState, misbehaviour)
      // clientState.mspInfos[misbehaviour.mspID].froozen = true
      break
    default:
      assert(false)
      break
  }
  // set("clients/{identifier}", clientState)
}

function checkMisbehaviourEndorsedCommitmentAndUpdateState(
  clientState: ClientState, misbehaviour: MisbehaviourEndorsedCommitment) {
  // TODO: detect contradiction about state transition in misbehaviour
}

function checkMisbehaviourForChaincodeHeaderSequence(clientState: ClientState, misbehaviour: MisbehaviourSubHeader) {
  assert(typeof misbehaviour.h1 == "ChaincodeHeader")
  assert(typeof misbehaviour.h2 == "ChaincodeHeader")
  ch1 = misbehaviour.h1 as ChaincodeHeader
  ch2 = misbehaviour.h2 as ChaincodeHeader
  assert(misbehaviour.sequence == ch1.sequence.value)
  assert(ch1.sequence == ch2.sequence)

  assert(ch1.proof.nsIndex == ch2.proof.nsIndex)
  assert(ch1.proof.writeSetIndex == ch2.proof.writeSetIndex)
  assert(ch1.proof.proposal != ch2.proof.proposal)
  assert(ch1.proof.signatures != ch2.proof.signatures)

  assert(verifyChaincodeHeader(clientState, ch1))
  assert(verifyChaincodeHeader(clientState, ch2))
}

function checkMisbeaviourForPolicySequence(clientState: ClientState, misbehaviour: MisbehaviourSubHeader) {
  assert(misbehaviour.sequence == getSequence(misbehaviour.h1))
  assert(getSequence(misbehaviour.h1) == getSequence(misbehaviour.h2))

  assert(misbehaviour.h1.GetSignBytes() != misbehaviour.h2.GetSignBytes())

  assert(verifySubHeader(clientState, misbehaviour.h1))
  assert(verifySubHeader(clientState, misbehaviour.h2))
}

// check a misbehaviour on updating the config of a MSP
function checkMisbehaviourForMSPInfoSequence(clientState: ClientState, misbehaviour: MisbehaviourSubHeader) {
  assert(typeof misbehaviour.h1 == "MSPInfoHeader")
  assert(typeof misbehaviour.h2 == "MSPInfoHeader")
  mh1 = misbehaviour.h1 as MSPInfoHeader
  mh2 = misbehaviour.h2 as MSPInfoHeader

  assert(misbehaviour.sequence == mh1.policy.sequence)
  assert(mh1.policy.sequence == mh2.policy.sequence)

  assert(mh1.mspID == mh2.mspID)
  assert(mh1.ibcPolicySequence == mh2.ibcPolicySequence)
  assert(mh1.config != mh2.config)
  assert(mh1.signatures != mh2.signatures)

  assert(verifyMSPInfoHeader(clientState, mh1))
  assert(verifyMSPInfoHeader(clientState, mh2))
}

function verifySubHeader(clientState: ClientState, subHeader: SubHeader) {
  switch(typeof subHeader) {
  case "ChaincodeHeader":
    return verifyChaincodeHeader(clientState, subHeader)
  case "PolicyHeader":
    return verifyPolicyHeader(clientState, subHeader)
  case "ChaincodeInfoHeader":
    return verifyChaincodeInfoHeader(clientState, subHeader)
  case "MSPInfoHeader":
    return verifyMSPInfoHeader(clientState, subHeader)
  default:
    assert(false)
  }
}
```
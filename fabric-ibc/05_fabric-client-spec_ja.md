# Fabric-client Spec

Fabric-clientは、[ICS-02](https://github.com/cosmos/ics/tree/master/spec/ics-002-client-semantics)を満たすHyperledger fabric用のclientである。

## ClientState

```go
type ClientState struct {
    ID                  string          `json:"id" yaml:"id"`
    LastChaincodeHeader ChaincodeHeader `json:"last_chaincode_header" yaml:"last_chaincode_header"`
    LastChaincodeInfo   ChaincodeInfo   `json:"last_chaincode_info" yaml:"last_chaincode_info"`
}
```

ClientStateは、Endorsement policy, ChaincodeのIDおよびVersionをtrackする。

## ConsensusState

```go
type ConsensusState struct {
    Timestamp time.Time
    Height    uint64
}
```

Fabric clientは、EndorserがIBC chaincode上で管理するSequenceとTimestampをtrackする。


## Header

HeaderはChaincodeHeaderとChaincodeInfoのいずれか、もしくは両方を含む。

```go
type Header struct {
    ChaincodeHeader *ChaincodeHeader `protobuf:"bytes,1,opt,name=ChaincodeHeader,proto3" json:"ChaincodeHeader,omitempty"`
    ChaincodeInfo   *ChaincodeInfo   `protobuf:"bytes,2,opt,name=ChaincodeInfo,proto3" json:"ChaincodeInfo,omitempty"`
}

type ChaincodeHeader struct {
    Sequence commitment.Sequence `protobuf:"bytes,1,opt,name=sequence,proto3" json:"sequence"`
    Proof    CommitmentProof     `protobuf:"bytes,2,opt,name=proof,proto3" json:"proof"`
}

type Sequence struct {
    Value     uint64 `protobuf:"varint,1,opt,name=value,proto3" json:"value"`
    Timestamp int64  `protobuf:"varint,2,opt,name=timestamp,proto3" json:"timestamp"`
}

type ChaincodeInfo struct {
    ChannelId         string        `protobuf:"bytes,1,opt,name=channel_id,json=channelId,proto3" json:"channel_id,omitempty"`
    ChaincodeId       ChaincodeID   `protobuf:"bytes,2,opt,name=chaincode_id,json=chaincodeId,proto3" json:"chaincode_id"`
    EndorsementPolicy []byte        `protobuf:"bytes,3,opt,name=endorsement_policy,json=endorsementPolicy,proto3" json:"endorsement_policy,omitempty"`
    IbcPolicy         []byte        `protobuf:"bytes,4,opt,name=ibc_policy,json=ibcPolicy,proto3" json:"ibc_policy,omitempty"`
    Proof             *MessageProof `protobuf:"bytes,5,opt,name=proof,proto3" json:"proof,omitempty"`
}
```

### ChaincodeHeader.Proof

ConsensusState内のSequeneを更新するために必要となる、fabric clientによる提案の結果への証明。含まれる署名は、ClientStateが保持するEndorsement Policyを満たす必要がある。

```go
type CommitmentProof struct {
    Proposal      []byte   `protobuf:"bytes,1,opt,name=proposal,proto3" json:"proposal,omitempty"`
    NsIndex       uint32   `protobuf:"varint,2,opt,name=ns_index,json=nsIndex,proto3" json:"ns_index,omitempty"`
    WriteSetIndex uint32   `protobuf:"varint,3,opt,name=write_set_index,json=writeSetIndex,proto3" json:"write_set_index,omitempty"`
    Identities    [][]byte `protobuf:"bytes,4,rep,name=identities,proto3" json:"identities,omitempty"`
    Signatures    [][]byte `protobuf:"bytes,5,rep,name=signatures,proto3" json:"signatures,omitempty"`
}
```

### ChaincodeInfo.Proof

ClientState内のChaincodeInfoを更新するために必要となる、ChaincodeInfoへの証明。含まれる署名は、ClientStateが保持するIBC Policyを満たす必要がある。

```go
type MessageProof struct {
    Identities [][]byte `protobuf:"bytes,1,rep,name=identities,proto3" json:"identities,omitempty"`
    Signatures [][]byte `protobuf:"bytes,2,rep,name=signatures,proto3" json:"signatures,omitempty"`
}
```

## Validity predicate

この関数は、与えられたHeaderが現在登録されているEndorsement policyに従ってHeaderに各署名がついていることを確認する。

```go
func checkValidity(
    clientState types.ClientState, header types.Header, currentTimestamp time.Time,
) error {
    if header.ChaincodeHeader == nil && header.ChaincodeInfo == nil {
        return errors.New("either ChaincodeHeader or ChaincodeInfo must be non-nil value")
    }
    if header.ChaincodeHeader != nil {
        // assert header timestamp is past latest clientstate timestamp
        if header.ChaincodeHeader.Timestamp.UnixNano() <= clientState.GetLatestTimestamp().UnixNano() {
            return sdkerrors.Wrapf(
                clienttypes.ErrInvalidHeader,
                "header blocktime ≤ latest client state block time (%s ≤ %s)",
                header.ChaincodeHeader.Timestamp.String(), clientState.GetLatestTimestamp().String(),
            )
        }
        if header.ChaincodeHeader.Sequence != int64(clientState.GetLatestHeight()+1) {
            return sdkerrors.Wrapf(
                clienttypes.ErrInvalidHeader,
                "header sequence != expected client state sequence (%d != %d)", header.ChaincodeHeader.Sequence, clientState.GetLatestHeight()+1,
            )
        }
        if err := types.VerifyChaincodeHeader(clientState, *header.ChaincodeHeader); err != nil {
            return sdkerrors.Wrap(
                clienttypes.ErrInvalidHeader,
                err.Error(),
            )
        }
    }
    if header.ChaincodeInfo != nil {
        if len(header.ChaincodeInfo.PolicyBytes) == 0 {
            return sdkerrors.Wrapf(
                clienttypes.ErrInvalidHeader,
                "ChaincodeInfo.PolicyBytes must be non-empty value",
            )
        }
        if err := types.VerifyChaincodeInfo(clientState, *header.ChaincodeInfo); err != nil {
            return sdkerrors.Wrap(
                clienttypes.ErrInvalidHeader,
                err.Error(),
            )
        }
    }
    return nil
}
```

HeaderにはChaincodeHeaderとChaincodeInfoのいずれかもしくは両方の要素が含まれる。これらに対して、それぞれ次の検証関数で確認する。


### VerifyChaincodeHeader

ChaincodeHeaderをWriteSetに含むProposalResponseに対して、現在登録されているEndorsement policyに従ってsignされていることを確認する。

```go
// VerifyChaincodeHeader verifies ChaincodeHeader with last Endorsement Policy
func VerifyChaincodeHeader(clientState ClientState, h ChaincodeHeader) error {
    lastci := clientState.LastChaincodeInfo
    ok, err := VerifyEndorsedCommitment(clientState.LastChaincodeInfo.GetFabricChaincodeID(), lastci.EndorsementPolicy, h.Proof, MakeSequenceCommitmentEntryKey(h.Sequence.Value), h.Sequence.Bytes())
    if err != nil {
        return err
    } else if !ok {
        return errors.New("failed to verify a endorsed commitment")
    }
    return nil
}

// VerifyEndorsedCommitment verifies a key-value entry with a policy
func VerifyEndorsedCommitment(ccID peer.ChaincodeID, policyBytes []byte, proof CommitmentProof, key string, value []byte) (bool, error) {
    policy, err := getPolicyEvaluator(policyBytes, GetConfig())
    if err != nil {
        return false, err
    }
    if err := policy.EvaluateSignedData(proof.ToSignedData()); err != nil {
        return false, err
    }
    id, rwset, err := getTxReadWriteSetFromProposalResponsePayload(proof.Proposal)
    if err != nil {
        return false, err
    }
    if !equalChaincodeID(ccID, *id) {
        return false, fmt.Errorf("got unexpected chaincodeID: expected=%v actual=%v", ccID, *id)
    }
    return ensureWriteSetIncludesCommitment(rwset.NsRwSets, proof.NsIndex, proof.WriteSetIndex, key, value)
}
```

### VerifyChaincodeInfo

現在登録されているIBC policyに従ってChaincodeInfoに対してsignされていることを確認する。


```go
// VerifyChaincodeInfo verifies ChaincodeInfo with last IBC Policy
func VerifyChaincodeInfo(clientState ClientState, info ChaincodeInfo) error {
    if info.Proof == nil {
        return errors.New("a proof is empty")
    }
    return VerifyEndorsedMessage(clientState.LastChaincodeInfo.IbcPolicy, *info.Proof, info.GetSignBytes())
}

// VerifyEndorsedMessage verifies a value with given policy
func VerifyEndorsedMessage(policyBytes []byte, proof MessageProof, value []byte) error {
    policy, err := getPolicyEvaluator(policyBytes, GetConfig())
    if err != nil {
        return err
    }
    sigs := makeSignedDataListWithMessageProof(proof, value)
    if err := policy.EvaluateSignedData(sigs); err != nil {
        return err
    }
    return nil
}
```

## State verification functions

Stateの各種検証関数は、Endorserが生成するEndorsed commitment(EC)をProofとして与えることにより、検証処理を行う。
ECは、Fabric networkのclientにより、Endorsed commitment generator(ECG)を提供するchaincodeを実行するtransaction proposalをEndorserに送信することによって得られる。

### VerifyClientConsensusState

対象のBlockchain上で個別のClientに保持されるConsensusStateへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyConnectionState

対象のBlockchain上に保持される個別のConnectionStateへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyChannelState

対象のBlockchain上に保持される個別のChannelStateへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketCommitment

特定のChannel、Port、Sequence時点で発信されるPacketへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, commitmentBytes); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketAcknowledgement

特定のChannel、Port、Sequence時点で受信されるPacket AckへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyPacketAcknowledgementAbsence

特定のChannel、Port、Sequence時点で受信されるべきPacket Ackが欠けていることへのProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, []byte{}); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

### VerifyNextSequenceRecv

特定のChannel、Portで受信されるべき次のSequenceに対するProofを検証する。

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
    if ok, err := VerifyEndorsement(cs.LastChaincodeInfo.GetFabricChaincodeID(), cs.LastChaincodeInfo.EndorsementPolicy, fabProof, key, bz); err != nil {
        return err
    } else if !ok {
        return fmt.Errorf("unexpected value")
    }
    return nil
}
```

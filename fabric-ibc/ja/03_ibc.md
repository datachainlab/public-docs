# IBC

IBC ProtocolはInter-Blockchain Communication Protocolの事で、特に本文書ではCosmosがInterchain Standards（以降、ICS）として規格策定を推進しているIBC Protocol（以降、IBC）を指す。

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
    https://github.com/cosmos/ibc/blob/old/ibc/2_IBC_ARCHITECTURE.md#diagram

CosmosのIBCに関する詳細は、[github上のリポジトリ](https://github.com/cosmos/ibc) を参照のこと。一部の仕様については[日本語訳](https://github.com/cosmos/ibc/tree/old/translation/ja)も存在する。

以降では、特にFabric IBCがIBCを実装するにあたって必要となる仕様について言及する。

## Client

ClientはIBC上の[ICS-002](https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics)で定義されたlight clientを指す。
Clientの目的は、IBCで通信するBlockchainが、他方のBlockchainで合意されたStateの更新を検証できるようにすることである。

Clientの更新はHeaderの提出によって行われる。提出されたHeaderの検証が行われた後、内部で保持するConsensusStateとClientStateを更新する。

### ConsensusState

ConsensusStateは、後述のValidity Predicateによって、Headerの検証に用いられる。
https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#consensusstate

Fabric-IBCでの定義については[ConsensusState](05_fabric-client-spec.md#conensusstate)を参照。

### ClientState

ClientStateはあるHeightで特定のキー/値ペアがState内に存在する、あるいは存在していないことのProofを検証するために用いられる。
https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#clientstate

Fabric-IBCでの定義については
[ClientState](05_fabric-client-spec.md#clientstate)を参照のこと。

### Header

HeaderはConsensusStateを更新するための情報を含む。
https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#header

Fabric-IBCでの定義については[Header](05_fabric-client-spec.md#header)を参照のこと。

### Validity Predicate

現在のConsensusStateに基づいてHeaderを検証する関数を指す。
https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#validity-predicate

Fabric-IBCでは、Headerを検証する際に、後述するEndorsed Commitmentの検証も行う。
内部では以下のような検証用関数を用いる。

#### VerifyChaincodeHeader
ChaincodeHeaderをWriteSetに含むProposalResponseに対して、現在登録されているEndorsement policyに従ってsignされていることを確認する。

#### VerifyChaincodeInfo
現在登録されているIBC policyに従ってChaincodeInfoに対してsignされていることを確認する。

 詳細については以下を参照のこと。
[Validity Predicate](05_fabric-client-spec.md#validity-predicate)

### State Verification Function

Clientが追跡すべきStateの内部状態を検証する関数を指す。

Fabric-IBCでは、Stateの検証をする際に、後述するEndorsed Commitmentの検証も行う。
次のような関数がICSによって[定義されている](https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#state-verification)。

#### verifyClientConsensusState
対象のBlockchain上で個別のClientに保持されるConsensusStateへのProofを検証する。

#### verifyConnectionState
対象のBlockchain上に保持される個別のConnectionStateへのProofを検証する。

#### verifyChannelState
対象のBlockchain上に保持される個別のIBC ChannelStateへのProofを検証する。

#### verifyPacketData
特定のIBC Channel、Port、Sequence時点で発信されるPacketへのProofを検証する。

#### verifyPacketAcknowledgement
特定のIBC Channel、Port、Sequence時点で受信されるPacket AckへのProofを検証する。

#### verifyPacketAcknowledgementAbsence
特定のIBC Channel、Port、Sequence時点で受信されるべきPacket Ackが欠けていることへのProofを検証する。

#### verifyNextSequenceRecv
特定のIBC Channel、Portで受信されるべき次のSequenceに対するProofを検証する。

Fabric-IBCでの詳細については以下を参照のこと。
[State Verification Functions](05_fabric-client-spec.md#state-verification-functions)

### Endorsed Commitment

IBCでは、あるキー/値ペアが通信相手のBlockchain上のStateに存在している（あるいは、存在していない）というCommitmentに対して、少ない計算コストで検証できるProofが必要になる。この詳細は[ICS-023](https://github.com/cosmos/ibc/tree/master/spec/core/ics-023-vector-commitments)で定義される。

Fabric-IBCでは、ClientState内で保持されるEndorsement Policyを満たすEndorserに対して、Read-Write Setに上記のペアが含まれるようなProposal Responseを返すChaincodeをQueryすることによってProofを作成する。これをEndorsed Commitmentと呼ぶことにする。

Endorsed Commitmentの検証は、Proposal Responseに対する署名が上記のClientState内Endorsement Policyを満たしていることを確認する。

Endorsed Commitmentを作成する用途として[Fabric IBC Modules](#fabric-ibc-modules)が提供される。

Endorsed Commitmentの詳細については、以下を参照のこと。
[Fabric Client Spec](05_fabric-client-spec.md)

## Connection, Channel

IBCにおいて、Connectionは[ICS-03](https://github.com/cosmos/ibc/tree/master/spec/core/ics-003-connection-semantics)、Channelは[ICS-04](https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics)で定義される。

Connectionは、IBCで通信する2つのBlockchain上でそれぞれが持つ状態で、Clientと関連付けて利用される。Blockchain間で通信するには、Connectionを確立する必要がある。

Channelは、Connectionを利用して、2つのBlockchain上の各Module間でMessageが配送されるためのセマンティクスを提供する。
それぞれの詳細については各ICSを参照のこと。

Fabric-IBCでも、Connection、Channelの状態遷移モデルはIBCに従う。
一方、それぞれの状態遷移時の検証には、Endorsed Commitmentを用いる。

Connection、Channelが開かれるまでのフロー図は以下を参照のこと。
[Connection、Channel確立](04_architecture.md#connectionchannel確立)

## Packet

[ICS-04](https://github.com/cosmos/ibc/tree/master/spec/core/ics-004-channel-and-packet-semantics)で定義される。
RelayerがBlockchain間でrelayするための情報として、送信元、宛先双方で使用するIBC ChannelやPortの情報とdata等を持つ。dataは、個別のModuleに実装されたロジックによって規定される。

## Relayer

Relayerは[ICS-018](https://github.com/cosmos/ibc/tree/master/spec/relayer/ics-018-relayer-algorithms)で定義される。
IBC上で、あるBlockchain上のトランザクションの状態を読み取り、Blockchain間でトランザクションをPacketとしてrelayする機能を持つオフチェーンプロセスのことである。

ICS上で[言及](https://github.com/cosmos/ibc/tree/master/spec/relayer/ics-018-relayer-algorithms#desired-properties)されるように、Relayerは以下のような性質を満たすことが望ましい。Fabric-IBCにおいても、これらは満たされる。

- Byzantineな振る舞いのRelayerがいる場合にも、IBCが満たすべき重要な安全性であるexactly-onceやdeliver-or-timeoutが損なわれないこと。
- 正常動作しているRelayerが1つあれば、Packetのrelayに関するlivenessが保たれること。
- Relay自体は誰でも行うことができ、Relayに伴って必要となる検証はすべてBlockchain上で実施されること。


### Relayerの初期化

Fabric-IBCでは、Relayerはサービスの起動前に、接続する2つのIBC Chaincodeに関連するConfigを与えられる。

#### Fabric上のIdentity ####
RelayerはFabricのclient機能を有するサービスである。そのため、以下の権限を満たすRoleをCAを用いて割当てる必要がある。

1. 送信元Blockchainに対してEventをQueryできる
2. 送信元BlockchainのStateに対してQueryできる
3. 双方のBlockchainに対してTransactionを提出できる

3.を満たす必要があるため、Relayerは両Blockchain上で必要なIdentityを割り当てられる必要がある。また、合わせて双方のlocal MSP情報をそれぞれのChaincodeと関連付けて管理する。

#### 両Chaincodeのpeerに対しての接続情報 ####
上記権限を満たすため、2つのChaincodeを運用するそれぞれのpeerに対する接続情報を管理する。

#### IBCの各種IDの情報 ####
これらの値はCosmos Relayerと[同様のもの](https://github.com/datachainlab/cross/blob/adbf051333acb1f8fbcbae6ff2888d477a4bfeb9/tests/demo/path01.json#L1)である。
両ChaincodeのclientID、connectionID、channelID、portなどを最初に設定しておく。

### Relayerとプライバシー

Fabric-IBCでは、Permissioned Blockchain上のStateを他のBlockchainにRelayするという点に関して、プライバシーの問題が起こり得る。これに関する整理は以下を参照のこと。

[RelayerのPrivacyに関して](06_appendix.md#relayerのprivacyに関して)

## Fabric-IBC Modules

Fabric clientが状態検証に必要とするProofやHeaderを生成する機能を提供するChaincode群である。以下の2つが提供される。

- Chaincode header generator
- Endorsed commitment generator
### Chaincode header generator

Chaincode header generator(CHG)は、Endorserが実行するChaincodeとして提供され、StateDBで現在のSequence値および生成時間を管理する。
以下の機能を提供する。

- 次のSequenceにインクリメントするAPI
- clientが指定する現在もしくは過去のSequenceに対してのEndorsed commitmentを生成する

### Endorsed commitment generator

Endorsed commitment generator(ECG)は、IBCのstateの特定のcommitmentを参照し、その存在を証明するProofとなるEndorser commitmentを生成する。
Endorserが実行するChaincodeとして提供され、State verificationのための関数ごとにAPIが用意される。


# Architecture

## ユーザ操作からの関連コンポーネント間フロー図

あるFabricネットワーク上のApplication Channel上で動作するChaincodeに対して、ユーザが更新処理を要求した際、その結果が別のFabricネットワーク上へ伝播する際のコンポーネント間のフローを整理する。

![](https://paper-attachments.dropbox.com/s_9444F561751AA4885E4D884B1952F8ACD65F9552197DAF0081552C2C836D7B24_1599820109175_file)


### IBC App

IBC moduleを管理し、入力をHandleし適切なmoduleにrouting及び実行し、状態を管理する。
Chaincode instanceはこのAppを通してIBCの機能を提供する。

### Fabric IBC Modules

Fabric clientが状態検証に必要とするProofやBlock headerを生成する機能を提供するchaincode群。
詳細については[Fabric IBC Modules](03_ibc_ja.md#fabric-ibc-modules)を参照。

### ConfigUpdater

Fabric CAの状態に変更があった場合、最新の状態をUpdateClientによりClientStateに反映するためのTransactionを提出する。

# Relayer シーケンス図

## Connection、Channel確立

[Connection, Channel](03_ibc_ja.md#connection-channel)にもあるように、Fabric-IBCでのConnection、Channelの状態遷移モデルはIBCに従う。一方、それぞれのフローではEndorsed Commitmentを用いた検証が必要となる。

2つのApplication Channel間でConnectionとChannelが開かれるまでのフローを以下のシーケンス図で示す。


![](https://paper-attachments.dropbox.com/s_9444F561751AA4885E4D884B1952F8ACD65F9552197DAF0081552C2C836D7B24_1596090739205_image.png)


上記のシーケンス図中に登場する内容について、Endorsed Commitmentについては、[Endorsed Commitment](03_ibc_ja.md#endorsed-commitment)を参照。

Verifyから始まる検証系の関数については、[State Verification Function](05_fabric-client-spec_ja.md#state-verification-functions)を参照。


## Client更新のフロー

2つのApplication Channel間でConnection、Channelが確立した後でのClient更新処理のフローをシーケンス図で示す。
ここではSequenceの更新を例に挙げるが、[Client](03_ibc_ja.md#client)が保持する状態の更新に伴って行われる処理である。IBC Policyの更新などでも必要となる。

![](https://paper-attachments.dropbox.com/s_9444F561751AA4885E4D884B1952F8ACD65F9552197DAF0081552C2C836D7B24_1596193073632_image.png)


## Endorsed Commitment の作成と検証

Fabric-IBCではあるApplication Channel上のChaincodeのStateにキー/値ペアが存在しているというCommitmentへのProofとして、[Endorsed Commitment](03_ibc_ja.md#endorsed-commitment)を用いる。

例として、2つの異なるApplication Channelがあるとき、一方(Channel1とする)上のStateに対してCommitment Proofを要求した後、他方(Channel2とする)でそのStateの検証を行う流れを、PacketCommitmentに関するシーケンス図で示す。

![](https://paper-attachments.dropbox.com/s_DAFCB926FADC8E47B8C76636D08E99EB63AB4DB6416B0C3DD9C48F433DE02567_1595848602931_image.png)


注意: 上図の灰色で覆われた領域(Chaincode)内の処理は1つのPeer上で起こる処理であり、内部のコンポーネント間ではネットワーク通信を伴わない。

Relayerはこの後、Channel2からChannel1に対して同様の手順でPacketAcknowledgementをrelayすることになる。

**HandleIBCTx**
Chaincode内のメソッドとして定義される。Msgを受け取り、初期化したIBC Appインスタンスに渡す。その後IBC Appの処理結果が正常であればそこに含まれるEventをFabricの[Event API](https://hyperledger-fabric.readthedocs.io/en/release-2.2/developapps/transactioncontext.html#stub)経由でResponseに含める。

**Msg**
Stateの更新を目的として、各Moduleで定義されるオブジェクト。
IBC Appによってroutingされる。

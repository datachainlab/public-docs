# Cross Framework

# Introduction

Cross Frameworkは、複数の異なるBlockchainに分散したデータの参照や機能の実行を行う、Cross-chain smart contract の開発を可能にするフレームワークである。このフレームワークを導入することにより、信頼されたシステムの介在なく、安全性を保証するための複雑なプロトコル設計・開発のコストを低減することが可能となる。それにより複数のBlockchain間でさらに多様な連携を安全性を保証しつつ実現させることができる。


## Smart contract

Smart contractとは、Blockchain上で実行可能なプログラムである。それらは状態をもち、多くの場合提供される機能は関数として公開され、定められた条件に従い実行可能である。ユーザが生成したTransactionによって実行され、状態の更新を行う。EthereumやHyperledger fabricなど多くのブロックチェーンにおいて、単一のChain or Channelの処理を行うトランザクションのみがサポートされており、他のブロックチェーンとのInteroperabilityは考慮されていないものが多い。

Smart contractの記述言語は、ブロックチェーンやDLTごとに様々なものがサポートされており、Cross Frameworkにおいては現在Go言語のサポートがされている。


## BlockchainのInteroperability

Blockchainは、一般に他BlockchainとのInteroperabilityの能力は乏しく、Transactionのサポートも単一のChain上の処理に特化しており、別のシステムの処理を含むトランザクションのサポートはされていないことが多い。それにより、各ブロックチェーンで形成されたネットワークのサイロ化が起こり、それぞれで立ち上がったエコシステムの発展を制限してしまうことになる。また、ブロックチェーンシステムのスケーリング面においても、(ブロックチェーンによる特性を保つためには)連携するアプリケーションは同一のブロックチェーン上にある必要性があることが、厳しい制約となるケースがある。


## Inter-Blockchain Communication(IBC)

IBCとは、Cosmosを中心として開発が進められているBlockchain間のコミュニケーションを可能にするInteroperabilityのためのプロトコルである。プロトコルはICSとして標準化が進められている。

IBCは、以下のようなフローによる通信を行う。

![https://github.com/cosmos/ics/blob/master/ibc/2_IBC_ARCHITECTURE.md#diagram](https://paper-attachments.dropbox.com/s_BF6A6C558FB10E2A2F4E74E9F7B342EF6228422735BC5F474C1D1BF9C0273659_1596156271353_Screenshot+from+2020-07-31+09-44-07.png)


各用語はICSの[Github](https://github.com/cosmos/ics) repositoryにて詳細な定義、説明がある。


- **Client**
> この規格は、IBC プロトコルを実装する Machine の Consensus アルゴリズムが満たすべき特性を規定します。これらの特性はより上位レベルのプロトコルの抽象化において効率的で安全な検証を行うために必要となります。IBC では、他の Machine の consensus 転写物 と state のサブコンポーネントを検証するために利用されるアルゴリズムを「検証用関数（validity predicate）」と呼び、検証者が正しいと仮定した状態とペアにすることで「light client」（しばしば「client」と短縮される）を形成します。


- **Connection**
> connection は2つの chain 上の永続的なデータ構造で、接続中の他方の台帳の consensus state に関する情報を含んでいます。一方の chain の consensus state を更新すると、もう一方の chain の connection オブジェクトの state が変化します。


- **Channel**
> channel はメタデータを含む2つの chain 上の永続的なデータ構造で、packet の順序付け、正確に1回限りの配信、およびリプレイの防止を容易にします。channel を介して送信された packet は、その内部 state を変更します。channel は、多対1の関係で connection に関連付けられます。単一の connection には、任意の数の関連付けられた channel を含めることができます。すべての channel には、channel が作成されるより前に作成された単一の関連付けられた connection が必要になります。


- **Packet**
> packet は、シーケンス関連のメタデータ（ IBC 仕様で定義）と packet データと呼ばれる不透明な値フィールド（トークンの量や額面などのアプリケーション層で定義されたセマンティクス）を備えた個別のデータ構造です。packet は、特定の channel を介して（さらに、特定の connection を介して）送信されます。


- **Relayer process**
> relayer プロセスはオフチェーンプロセスで、2つ以上の machine 間で IBC packet データとメタデータを中継します。そのために machine の state をスキャンしてトランザクションを送信します。

フローの各ステップは、以下の通りである:


1. DLT *A* 上
    1. Module（アプリケーション固有）
    2. Handler（複数の ICS 間で部分的に定義されます）
    3. Packet（[ICS 4](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-004-channel-and-packet-semantics) で定義されます）
    4. Channel（[ICS 4](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-004-channel-and-packet-semantics) で定義されます）
    5. Connection（[ICS 3](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-003-connection-semantics) で定義されます）
    6. Client（[ICS 2](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-002-client-semantics) で定義されます）
    7. Consensus（送信 packet を含むトランザクションを確定します）
2. オフチェーン
    1. Relayer（[ICS 18](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-018-relayer-algorithms) で定義されます）
3. DLT *B* 上
    1. Consensus（受信 packet を含むトランザクションを確定します）
    2. Client（[ICS 2](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-002-client-semantics) で定義されます）
    3. Connection（[ICS 3](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-003-connection-semantics) で定義されます）
    4. Channel（[ICS 4](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-004-channel-and-packet-semantics) で定義されます）
    5. Packet（[ICS 4](https://github.com/cosmos/ics/blob/master/translation/ja/spec/ics-004-channel-and-packet-semantics) で定義されます）
    6. Handler（複数の ICS 間で部分的に定義されます）
    7. Module（アプリケーション固有）

Cross Frameworkでは、IBCのChannelとPacketを用いたBlockchain間の通信を利用し、Cross-chain Transactionsを実現している。つまり、上図における、Module A, BがCross Frameworkが提供するModuleや開発者がデプロイするSmart contractを含むModuleに相当する。


## Cross-chain smart contract

Cross-chain smart contractとは、ICS-04のChannelで接続された複数の異なるChainで提供されるSmart contractの機能や状態を相互に呼び出し、参照を行うなどのCross-chain callを行うSmart contractのことである。Cross-chain smart contractにより、より複雑な状態を持つアセットのChain間転送やスワップを実現可能となることに加え、新しくこのようなプロトコルを開発者が実装する場合にも安全に最小限のロジックの開発に集中できるようになる。

また、このようなSmart contractを実行するTransactionをCross-chain Transactionsと呼ぶ。これは複数の異なるブロックチェーン間で行う分散トランザクションを同義である。そのため、分散トランザクション一般と同じく、Cross-chain Transactionsは、ACID特性で定義されているように適切なトランザクションの安全性を保証する必要がある。これらの安全性の保証をどのように実現しているかは[Cross-chain Transactionsの章](#cross-chain-transactions)で説明する。


## Cross Framework

Cross Frameworkは、Cross-chain Smart Contractの開発およびそれらに対するCross-chain Transactionsのサポートを可能にするFrameworkである。

フレームワークは、Cross-chain transactionsをサポートするためのCross ModuleとSmart contractを提供するContract Moduleから構成される。Cross Moduleは、Cross-chain transactionsを実現するために必要なModuleである。これはIBC Moduleを介した他のChainとのAtomic commitのためのプロトコルを実装するCross Handlerとそれらの状態を保持するStoreから構成される。Contract Moduleは、Smart Contractを提供するContract handler、それらの状態を保持するState storeから構成される。

![Fig. Architecture](https://paper-attachments.dropbox.com/s_BF6A6C558FB10E2A2F4E74E9F7B342EF6228422735BC5F474C1D1BF9C0273659_1599547501386_Screenshot+from+2020-09-08+15-44-26.png)


Cross Frameworkを使ったSmart contractを開発するには、開発者はContract Module内のContract Handlerを実装する必要がある。この詳細は[Contract handlerとState store](#contract-handlerとstate-store)で説明している。

Cross Frameworkは、各ブロックチェーンの実装に依存しない機能の提供を目指している。現在すでに、TendermintとHyperledger fabricについてサポートしており、今後もサポート範囲を広げていくことを予定している。


# Cross-chain smart contract


## Smart contract

Cross Frameworkにおいて、Smart contractは、開発者がリクエストに応じて動的にコントラクトを呼び出し可能となるようなContract Handlerの実装によりもたらされる。それぞれの機能はContract関数として定義される。Cross-chain smart contractは、この複数のchainのContract関数の呼び出しを含むSmart contractを指す。また、これにより定義されたSmart contractのContract関数は、通常の単一chainでの実行と後述するCross-chain Transactionによる実行の両方で同じプログラムが実行される。


## Contract handlerとState store

Contract handlerには、複数のContract関数を実装可能である。各Contract関数には、Smart contractの状態を管理するためのState storeが与えられる。各関数は呼び出し引数や記述したロジックに従い、決定性のある状態遷移を行い、その状態をこのState storeに永続化する。

State storeはGet(K), Set(K,V), Delete(K)といった一般的なKey-Value Storeの操作をサポートする。これらはCosmos-SDKのKVStoreとほとんど違いはない。しかし、参照する状態と競合する処理中のCross-chain Transactionがある場合は別である。State storeは、Getしようとするキーに対してロックが取られていないことを確認する。ロックがすでに存在する場合、この単一Chainで呼び出されたSmart contractの実行は失敗する。このState storeへの操作とロックの詳細については、Cross-chain Transactionsの[State storeとLocking mechanism](#state-storeとlocking-mechanism)で説明している。

以下は、UserIDごとにトークン残高を管理するようなコントラクトにおけるTransfer関数の擬似言語による実装例である。

```go
func Transfer(ctx contract.Context, store cross.Store) {
    args := ctx.Args()
    fromID, toID, amount := args[0], args[1], args[2]
    assert(ctx.Signer == fromID)
    fromBalance := store.Get(fromID)
    assert(fromBalance >= amount)
    toBalance := store.Get(toID)
    store.Set(toID, toBalance+amount)
    store.Set(fromID, fromBalance-amount)
}
```

## Cross-chain calls

各Chainにおいて、Contract handlerで定義されたContract関数は、それぞれ別のChainのContract関数を呼び出し可能である。これをCross-chain callsと呼んでいる。これにより、Cross-chain Transactionsにおいて、単に複数の関数をAtomicに実行できるだけでなく、Chain間の価値の移転などのより複雑なプロトコルを含むSmart contractを開発可能にしている。

以下にChain間のトークンの移転のコントラクトの例を示す。

```go
// Chain A
func Peg(ctx contract.Context, store cross.Store) {
    fromID, amount, toID := ctx.Args()[0], ctx.Args()[1], ctx.Args()[2]
    ret := CallContract(ctx, contract.NewContractCallInfo(ChainB::Lock, fromID, amount), []contract.ID{ctx.Signer})
    assert(amount == ret)
    toBalance := store.Get(toID)
    store.Set(toID, toBalance+amount)
}
```

```go    
// Chain B
func Lock(ctx contract.Context, store cross.Store) {
    fromID, amount := ctx.Args()[0], ctx.Args()[1]
    fromBalance := store.Get(fromID)
    assert(fromBalance >= amount)
    store.Set(fromID, fromBalance-amount)
    return amount
}
```

上記例において、Cross-chain callsを利用して、Peg関数からLock関数を呼び出しを行い、Lock関数の戻り値を取得し、その値を用いて残りの手続きを実行している。このように同一Chainの他のContract関数を呼び出すように別Chainの関数を呼び出すことが可能である。これにより、Chain間の資産の移転やChainをまたいだAtomic swapを複雑な状態や条件を付与しつつも実装可能となっている。このようなCross-chain smart contractを実行するには、Cross-chain transactionを実行する必要がある。その詳細は[Cross-chain Transactionsの章](#cross-chain-transactions)で説明する。


# Cross-chain transactions


## What is Cross-chain transactions?

Cross-chain Transactionは、IBC channelで接続されたChain間で実行可能な分散Transactionである。Cross Frameworkでは、Transactionの安全性を保証するために複数のAtomic commit protocolをサポートし、またそれを実現するためのLock mechanismを実装したState storeを提供している。

Transactionはユーザからリクエストを受けたChainがその内容を検証し、指定されたChainに対してContract呼び出しリクエストを含むPacketを送信することで開始される。ユーザのTransaction開始のリクエストを受け取るChainをInitiator chainと呼ぶ。Initiator chainは、リクエストを受け取ると、Atomic commitのフローを指定された形式に従い実行する。例えば、Atomic commitとしてTwo phase commitを選択した場合、Initiator chainが2PCのCoordinatorの役割を担い、ユーザのTransactionリクエストに含まれる各ChainのContract関数の呼び出し、およびその結果を管理し、最終的なCommitの可否を決定し、その決定を受け取った各Participant chainsがCommitもしくはAbortを行う。


## Cross-chain transactionの作成と検証

Cross-chain transactionを開始するために、ユーザは以下の要素を持つ`MsgInitiate`を作成し、Initiator chainに提出する。


- **Type**: Commit flowの種別。各Commit flowは[Atomic commit protocol](#atomic-commit-protocol)で説明している。
- **Nonce**: ナンス値
- **Timeout**: このTransactionが取り込むことが可能なBlock heightもしくはBlock timeの上限値
- **ContractTransactions**: 各Contractの呼び出し情報。以下の要素からなる構造体の配列となっている
    - **ChainID**: この`ContractTransaction`を処理するChainを示すID
    - **Signers**: このContractのTransaction作成に承認が必要となるAccountの配列。
    - **CallInfo**: Contractの識別子、関数名、引数などを含む呼び出し情報。フォーマットはContract handlerの仕様に依存する
    - **StateConstraint**: このContractが参照する状態、更新後の状態に対して制約をかけることができる。(オプション)詳細は、**State constraintsとSimulation**を参照
    - **ReturnValue**: このContractの実行の戻り値(オプション)
    - **Links**: このContractの実行時に参照される他のContract呼び出し結果(オプション)。詳細はLinkの項を参照。

Accountは、Transactionの実行者を定義する概念である。適切なAccountであるかをTransactionの内容に応じて検証される。Transactionの検証ロジック、およびAccountによる承認とその識別子の仕様はBlockchainやDLTごとに様々である。

Cross Frameworkでは、Transactionの開始の際に`Signers`で指定された全てのAccountの承認が必要であるが、各Accountの承認方法およびそれぞれの検証方法は定義されておらず、Fabric-IBCのStdTxによるClient Identifierの検証やCosmos-SDKのauth/StdTxの署名検証をBlockchainやDLTごとに利用する必要がある。

`MsgInitiate`を受け取ったInitiator chainは、承認が必要な全てのAccountにより、このCross-chain transactionが承認されたことを確認する必要がある。この検証フローにはいくつかのパターンがある。


1. Accountが鍵を持つ場合、Initiator chainに対し`MsgInitiate`の提出とともに各Accountによる署名を提出してそれを検証する。
   
2. Accountが鍵を持たない場合、Initiator chain上に提出された`MsgInitiate`の提出により発行されたTxを示すIDに対して、各Accountが非同期に承認する。

3. Accountが鍵を持たないかつInitiator chainにアクセスできない場合、各AccountはContract Moduleを含むChainに対して承認を行い、その結果をPacketを経由してInitiator chain上で検証する方法。これはPermissionedなchainに適したフローである。

上記のいずれかの方式で検証をした後にInitiator chainは、`Type`に従いAtomicなCommitを行うフローを開始する。`ContractTransactions`に指定された各ContractのTransactionを含めたPacketをそれぞれのChainのContract handlerに送信する。その際のCommitのフローの種類とそれぞれの詳細については、[Atomic commit protocolの章](#atomic-commit-protocol)で説明する。

Commitフローの種類にかかわらず、`MsgInitiate`で宣言された各Contractの関数の実行が全て成功し、また`StateConstraint`などの実行時の状態に対する制約がある場合はそれを満たす場合のみ全てのTransactionがCommitされることが保証されている。


## State constraintsとSimulation

State constraintsは、Transactionを実行する際の事前状態、事後状態、またはその両方に対して制約をかけるための機能である。この機能は、Contract関数の実行時に厳密な状態遷移を必要とする場合に役立つ。アプリケーションからContract関数を実行するTransactionをリクエストする際に、その処理を一度だけ実行することを厳密に保証することができる。

SimulationはTransactionの生成なしでContractの実行を行うことができる機能である。これを利用してState constraintsで指定するための制約を生成することが可能である。Simulateのリクエスト時に`StateConstraintType`を指定することで制約として参照可能なState storeの状態を返す。ユーザは`MsgInitiate`の作成時にこれをContractTransactionのStateConstraintに指定することでSimulation時の状態を利用した制約をTransactionにかけることができる。


## LinkとCross-chain calls

[Cross-chain calls](#cross-chain-calls)で紹介したような他のContract handlerの外部のContract関数を呼び出すには次のような要素を考慮する必要がある。

a. 各Contract HandlerのContract関数はそれぞれのChain上で並行して実行される場合がある

b. 外部のContract関数の実行はその呼び出し元の関数の実行とAtomicに行われる必要がある

c. 外部のContract関数の実行による返り値を呼び出し元関数で参照できる

このうち、a, bはAtomic commit protocol により実現される。Linkはこのcの要素を満たすための機能である。

Linkは以下の構造を持つ。

```go
type Link struct {
    SrcIndex uint8
}
```

`SrcIndex`により、呼び出し先のContract関数の実行を処理するTransactionを示す`ContractTransactions`のインデックスを指定する。

`MsgInitiate`を受け取ったInitiator chainは、`ContractTransactions`の各要素のLinkの情報を呼び出し先のContract関数のChainIDと返り値を含むオブジェクト `ConstantValueObject` に解決し、Linkの参照元のContract関数内の外部呼び出しはこの`ConstantValueObject`に置き換えられる。

以下のようなステップで実行される。


1. ContractHandlerの外部Contractを参照する関数は呼び出し情報に加え、参照先Chainを示す`ChainID`を設定する必要がある。これはChannelIDや[Interchain DNS](https://github.com/datachainlab/cosmos-sdk-interchain-dns)のDomain nameなどで実装される。

```go
type ChainID interface {
    Equal(ChainID) bool
}
```

2. Initiator chainは、`ContractTransactions`の各要素の`Links`の情報を元に呼び出し先のContract関数のChainIDと返り値を含むオブジェクト `ConstantValueObject` に解決する

```go
type ConstantValueObject struct {
    ChainID ChainID
    Value   []byte
}
```

3. Coordinator chainは、Atomic commit flowに従い、`ContractTransaction`と2の`ConstantValueObject`および参照先の呼び出し情報をPacketのDataとしてセットし、各ChainのContract handlerに送信する


4. ChainのContract handlerは、受け取ったPacketの呼び出し情報に従い、Contract関数を実行する。関数内の外部Contract関数の呼び出し時に、プログラム中の参照先のChainIDおよび呼び出し情報と比較し、一致している場合、`ConstantValueObject`の`Value`を返す。一致していない場合、その関数の処理は失敗となる。



# Atomic commit protocol

Atomic commit protocolとは、複数の操作の集合を1つの処理として実行可能にするプロトコルである。代表的なものとしては、[Two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)や[Three-phase commit](https://en.wikipedia.org/wiki/Three-phase_commit_protocol)などがある。

Cross Frameworkでは、現在、Simple commit protocol, Two-phase commit protocolの2種類のプロトコルをサポートしている。Simple protocolは参加者が2者に限られる。Two-phase commitは参加者が3者以上であってもサポートできるが、Simple protocolと比較するとフロー中のステップ数が多くなっており、つまり完了までの時間が長くなる。

各方式の詳細については、Two-phase commit protocol, Simple commit protocol をそれぞれ参照。


## Two-phase commit protocol

**プロトコルの概要**

[Two-phase commit(2PC)](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)では、あるChainをCoordinatorとし、他のContract関数の実行とコミットを行うChain群をParticipantsとして次のようなフローでコミット、もしくはアボート状態に至る。CoordinatorがParticipantsにコミット準備を要求し、すべてのParticipantsが成功したなら、全Participantsにコミット要求を出す。コミット準備に1人でも失敗したなら、全Participantsにアボート要求を出す。

**Commitのフロー**

このプロトコルをFig.1のようなフローで実現する。XはCoordinator, A, B, Cは各Participant, 矢印はフローの各ステップのリクエストの向きを示している。


![Two-phase commitのフロー](https://paper-attachments.dropbox.com/s_FFE17AA1F4B82D88109B81BA32CA19757CCAC2E7BBE2D24C9CAD3FFEE992E8B9_1581565269084_Screenshot+from+2020-02-13+12-40-55.png)



1. Initiate step


    - ユーザは、Transaction開始のリクエストである`MsgInitiate`をInitiator chainに提出する


    - Initiator chainは2PCのCoordinatorとして`MsgInitiate`に指定されたContractの関数を含むParticipant chainsに`Prepare`を要求するPacketである`PacketPrepare`を送信する



2. Prepare step


    - 各Participant chainsは、`PacketPrepare`を受け取ると、[Cross-chain-transactionの作成と検証](#cross-chain-transactionの作成と検証)の章で説明したContractTransactionで指定されるContract関数を実行する(この際、実際にコミットはされないことに注意する)。


    - 実行に成功した場合、State storeに対しての変更についてのロックを取得する。このロックメカニズムの詳細は、[State storeとLocking mechanism](#state-storeとlocking-mechanism)の章で説明している。最後にACK `PacketPrepareAcknowledgement`のStatusに成功を示す`PREPARE_RESULT_OK`をセットして返す


    - 実行に失敗した場合、変更操作を破棄して、ACK `PacketPrepareAcknowledgement` の`Status`に失敗を示す`PREPARE_RESULT_FAILED`をセットして返す



3. Confirm step


    - Coordinator chainは、Prepare stepにおいて各Participant chainが送信するACK `PacketPrepareAcknowledgement`を受け取る。各ACKの処理については、以下のような状態遷移を行う
        1. 次のACKの受信待ち
        2. 受信したACKの`Status`が`PREPARE_RESULT_OK`かつ未受信のACKがある場合、1.に遷移する。すべて受信した場合、各Participant chainにCommit要求をするためにCommit stepに進む
        3. 受信したACKの`Status`が`PREPARE_RESULT_FAILED`の場合、各Participant chainにAbort要求をするためにCommit stepに進む



4. Commit step


    - 各Participant chainsは、Commit要求があった場合、Prepare stepでロックしていた変更をState storeに適用し、ロックを削除してCoordinator chainに完了済みを示す`PacketCommitAcknowledgement`を送信する


    - 各Participant chainsは、Abort要求があった場合、Prepare stepでロックしていた変更とロックを削除してCoordinator chainに完了済みを示す`PacketCommitAcknowledgement`を送信する


**故障耐性について**

通常2PCのプロトコルは、Coordinatorの故障の場合にブロッキングプロトコルであることが知られている。Hung Dangらの[”Towards Scaling Blockchain Systems via Sharding”](https://www.comp.nus.edu.sg/~hungdang/papers/sharding.pdf)でも指摘されているようにCross-chain上でのAtomic commitの実現にあたり、以下のプロパティを満たす必要があると考える。

a. GeneralなTransactionに対してのsafety

b. 悪意のあるCoordinatorに対してのliveness

aについては、[State storeとLocking mechanism](#state-storeとlocking-mechanism)で提供されるロックメカニズムにより安全性が保たれる。

bについては、CoordinatorのState machineをBFT consensusを行うChain上で実行することにより保たれると考える。


## Simple commit protocol

**プロトコルの概要**

Simple commit protocolは、Participantの数に対する制約があるAtomic commitのprotocolである。Two phase commit protocolとは異なり、Participantは2つである必要があり、またそれらのどちらかがCoordinatorの役割を担う必要がある。

**Commitのフロー**


![Simple commit flow](https://paper-attachments.dropbox.com/s_BF6A6C558FB10E2A2F4E74E9F7B342EF6228422735BC5F474C1D1BF9C0273659_1597826874435_Screenshot+from+2020-08-19+17-47-27.png)



1. Initiate


    - ユーザは、Cross-chain transactionを開始するために`MsgInitiate`をInitiator chainに提出する


    - Simple commit flowにおいてInitiator chainはCoordinatorとParticipantのロールを兼ねる


2. Prepare(A)


    - Aは`MsgInitiate`で指定されたContract関数をPrepare実行する。


    - 実行に成功した場合、State storeに対しての変更についてのロックを取得する。このロックメカニズムの詳細は、[State storeとLocking mechanism](#state-storeとlocking-mechanism)の章で説明している。その後、BのContract関数の呼び出し情報を含む、Packet `PacketDataCall`を作成しBとのChannelに送信する。


    - 実行に失敗した場合、このTransactionの処理をAbortして終了する。


3. Commit(B)


    - Bは`PacketDataCall`を受け取り、指定されたContract関数を実行する。
    
    - 実行に成功した場合、Commitを行う。その後、`PacketCallAcknowledgement`にstatusとして`COMMIT_OK`をセットして、AとのChannelに送信する。
    
    - 実行に失敗した場合、Abortを行う。その後、`PacketCallAcknowledgement`にstatusとして`COMMIT_FAILED`をセットして、AとのChannelに送信する。


4. Commit(A)


    - Aは`PacketCallAcknowledgement`を受け取り、そのstatusを確認する。
    
    - statusが`COMMIT_OK`の場合、AはPrepare時の操作をCommitする。
    
    - statusが`COMMIT_FAILED`の場合、AはPrepare時の操作を破棄してAbortする。



## State storeとLocking mechanism

State storeは、[Smart contractの章](#cross-chain-smart-contract-1)で説明したように、Smart contractの状態を永続化するKey-value storeである。以下のような操作をサポートする。


- Get(K): `K`で指定したKeyに対応するValueを返す
- Set(K,V): `K`で指定したKeyに対して`V`で指定したValueをセットする
- Delete(K): `K`で指定したKeyと対応するValueを削除する

単一のChainにおいて逐次的にContract関数を実行する上では、これらの操作のたびに、もしくは関数の実行後にTransactionによる変更をStoreに反映すれば良いが、Transactionが処理するContract関数が複数chainに分散している場合、すべてのContractの状態遷移をAtomicに行う必要がある。利用するAtomic commit protocolにより異なるが、少なくとも1つのContract関数の実行はPrepare状態に遷移する必要がある。つまり、そのようなContractにおいては他の並行するTransactionと競合する可能性が発生するため、Transactionの[直列化可能性](https://en.wikipedia.org/wiki/Serializability)を保つための仕組みがState storeに必要となる。

Cross Frameworkでは、デフォルトのState storeの実装として各変更操作に対応するロックを取得する単純なStoreを提供している。このStoreは各変更操作に対して以下のようなロックを取得する。

| State storeへの操作 | 取得するロック  |
| --------------- | -------- |
| Get(K)          | なし       |
| Set(K, V)       | Kへの排他ロック |
| Delete(K)       | Kへの排他ロック |

Cross Frameworkでは、このようなState storeをTendermint/Cosmosに関しては、[KVStore](https://docs.cosmos.network/master/core/store.html)として提供し、Hyperledger Fabricに関しては、Chaincode上のStateへのoperationの[API](https://github.com/hyperledger/fabric-chaincode-go/blob/master/shim/interfaces.go#L28)を実装したものをwrapするKVStore互換のインタフェースを実装したStoreを提供する。

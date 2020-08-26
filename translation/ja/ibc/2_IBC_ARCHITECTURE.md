# 2: Inter-blockchain Communication プロトコルアーキテクチャ

**これは、IBC プロトコルに関する上位レベルのアーキテクチャとデータフローの概要です。**

**IBC 仕様に関する用語の定義は[こちら](./1_IBC_TERMINOLOGY.md)を参照してください。**

**大まかなプロトコル設計原則については、 [こちら](./3_IBC_DESIGN_PRINCIPLES.md)を参照してください。**

**ユースケース例については、[こちら](./4_IBC_USECASES.md)を参照してください。**

**デザインパターンの議論については、[こちら](./5_IBC_DESIGN_PATTERNS.md)を参照してください。**

このドキュメントでは、ブロックチェーン間通信（IBC）プロトコルスタックの認証、トランスポート、および順序付けレイヤーのアーキテクチャについて概説します。このドキュメントでは、個別のプロトコル詳細については説明していません。それらは個々の ICS に含まれています。

> 注： *台帳*、*chain*、および*ブロックチェーン*は、口語的な使用法に従い、このドキュメント全体で同じ意味で使用されています。

## IBC とは何か？

*IBC プロトコル*は、信頼性が高く安全な module 間の通信プロトコルです。ここで module とは独立した machine 群の上で動作する決定論的なプロセスです。machine 群には（「ブロックチェーン」や「分散台帳」のような）Replicated State Machine を含みます。

IBC は、信頼性が高く安全な module 間通信の上に構築されたあらゆるアプリケーションで使用することができます。アプリケーションの例としては、クロスチェーンアセット転送、アトミックスワップ、複数 chain を用いるスマートコントラクト（相互理解可能なVMの有無を問わない）、種々のデータやコードのシャーディング等が挙げられます。

## IBC とは何でないか？

IBC はアプリケーション層のプロトコルではありません。データ転送、認証、信頼性のみを取り扱います。

IBC は アトミックスワッププロトコルではありません。任意のクロスチェーンデータ転送や計算がサポートされています。

IBC は トークン転送プロトコルではありません。トークン転送は IBC プロトコルを用いたアプリケーション層の実例の1つです。

IBC はシャーディングプロトコルではありません。そこにあるのは複数 chain に分割された1つの state machine ではなく、共通インタフェースを共有した異なる chain 上に存在する多様な state machine の集合です。

IBC は Layer 2 スケーリングプロトコルではありません。IBC を実装する全ての chain は同じ「Layer」に存在します。ネットワークトポロジの中でそれぞれの chain は異なる点群を占有しているかもしれませんし、単一のroot chainや単一の validator セットは不要です。

## 動機

執筆時点で主流となっている2つのブロックチェーン、Bitcoin と Ethereum は現在それぞれ毎秒約7件、約20件のトランザクションをサポートしています。両者とも主にアーリーアダプターである熱狂的なユーザーが利用するのに留まっているにもかかわらず、最近は能力限界まで稼働しています。スループットはほとんどのブロックチェーンユースケースにとって成約であり、スループットの制限は分散 state machine の基本的な制約です。なぜならば、ネットワーク内のすべての（検証）ノードがすべてのトランザクションを処理する必要があるためです（現在のところこの仕様の範囲外ですが、将来のゼロ知識の解釈を含めて）。[Tendermint](https://github.com/tendermint/tendermint) などのより高速なコンセンサスアルゴリズムは、大きな定数倍でスループットを向上させる可能性がありますが、上記の理由により無制限にスケーリングすることはできません。分散台帳アプリケーションの幅広いデプロイを容易にするために必要となるトランザクションスループット、アプリケーションの多様性、コスト効率をサポートするには、実行とストレージを、並行動作できる多数の独立した consensus インスタンスに分割する必要があります。

設計の方向性の1つは、単一のプログラム可能な state machine を「シャード」と呼ばれる別個の chain に分散させ、並行的に実行して stateを分割したパーティションを格納することです。safety と liveness を推論し、シャード間でデータとコードを正しくルーティングするためには、これらの設計は「トップダウン・アプローチ」を取らなければなりません。つまり、単一のrootとなる台帳とシャードのスターまたはツリーを持つ特定のネットワークトポロジを構築し、そのトポロジを強制するためのプロトコル規則とインセンティブを設計しなければなりません。このアプローチは、シンプルさと予測可能性という利点がありますが、難しい[技術的](https://medium.com/nearprotocol/the-authoritative-guide-to-blockchain-sharding-part-1-1b53ed31e060)な[問題](https://medium.com/nearprotocol/unsolved-problems-in-blockchain-sharding-2327d6517f43)があり、すべてのシャードを単一の validatorセット（またはランダムに選ばれたサブセット）と単一の state machine または相互に理解可能なVMに固執させる必要があります。

さらに、単一の consensus アルゴリズム、state machine、sybil攻撃耐性は、必要な水準のセキュリティと汎用性を提供できない可能性があります。 consensus インスタンスは、サポートできる独立したオペレーター数に制限があります。つまり、consensus インスタンスによって守られている価値が増加するにつれて、特定のオペレーターが不正することによる償却利益が増加します。その一方、オペレータ不正に必要なコストは常に最も安価な方法が反映され（例：物理的な鍵の不正引き出しやソーシャルエンジニアリング）、無限に増加するわけではありません。単一のグローバルな state machine は、多様なアプリケーションの共通要素に対応する必要があるため、どのアプリケーションにとっても専用の state machine ほどには適さないものになります。単一の consensus インスタンスのオペレータは、自分達の特権的地位を乱用して、簡単に終了することを選択できないアプリケーションから超過利潤を得ようとする可能性があります。必要最小限の共通インターフェースのみを共有しながら、独立した consensus インスタンスと state machine が安全かつ自発的に相互作用できるメカニズムを構築することができれば、より望ましいでしょう。

*IBC プロトコル*は異なるアプローチをとり、スケーリングとインターオペラビリティの問題を違う形で定式化します。それにより見知らぬトポロジーに配置された異種混合の分散台帳ネットワークを安全かつ高い信頼性で相互運用することを可能にします。また可能な限り秘密を守り、その中で台帳はお互いにあるいは特定のトポロジーや state machine 設計から独立して多様化、発展、再配置することができます。chain 間インターオペラビリティの広範で動的なネットワークでは、散発的なビザンチン故障が予想されるため、プロトコルは、関係するアプリケーションや台帳の要件に従って、ビザンチン故障の潜在的なダメージを検出し、緩和し、封じ込めなければなりません。設計原則のより詳細な項目については、[こちら](./3_IBC_DESIGN_PRINCIPLES.md)を参照してください。

この異種混合のインターオペラビリティを容易にするために、IBC プロトコルは「ボトムアップ」アプローチを取り、2つの台帳間の相互運用を実装するために必要な要件、機能、特性を規定します。また、複数の相互運用台帳がより上位レベルのプロトコル要件を維持し、安全性/速度のトレードオフをさまざまに選択して構成する方法を規定します。このように、IBC はネットワークトポロジ全体について何も想定しておらず、何も必要としません。台帳の実装では既知の最小限の機能が利用可能かつ特性が満たされていれば良いだけです。実際、IBC 内の台帳は、それらのlight clientのconsensus 検証関数として定義されているため、「台帳」の範囲を拡大して、単一の machine や複雑な consensus アルゴリズムを含めることができます。

IBC は、エンドツーエンドの接続指向のステートフルプロトコルであり、個別の machine 上にある module 間での、信頼性があり順序付け可能な認証付きの通信を実現します。 IBC 実装は、ホスト state machine 上にある上位 module およびプロトコルと共存することを想定しています。 IBC をホストする state machine は、consensus 転写物の検証および暗号 commitment proof 生成用の機能を提供する必要があり、IBC packet relay（オフチェーンプロセス）は、state を片方の machine から読み取って他方に送信するために必要に応じてネットワークプロトコルおよび物理データリンクにアクセスできることが期待されます。

## Scope

IBC は、個別の machine 上の module 間で relay される構造化されたデータ packet の認証、転送、および順序付けを処理します。プロトコルは2つの machine 上の module 間で定義されますが、所定のトポロジーで接続された任意の数のmachine 上の任意の数の module 間で安全に同時に使用できるように設計されています。

## Interfaces

IBC は、一方では module  — スマートコントラクト や、他の state machine コンポーネント、または state machine 上の一部の独立したアプリケーションロジック部分 — に、  他方では consensus プロトコル、machine 、ネットワークインフラ（TCP/IP など）の間に配置されます。

IBC は module に対して、同じ state machine 上の別 module と対話するための機能を提供します。データ packet の送受信を確立されたconnection と channel (認証と順序付けのための基本要素。[定義](./1_IBC_TERMINOLOGY.md)を参照のこと。)上で行うための機能、connection や channel の開閉、選択や状態確認、packet 配送オプションの選択などが挙げられます。

IBC は、[ICS 2](../../spec/ics-002-client-semantics) で定義されている基本的な consensus プロトコルと machine の機能や特性を前提にしています。主だったものとして finality (あるいは閾値をもった finality gadget)、安価に検証可能な consensus 転写物、および単純な KVS 機能が挙げられます。ネットワーク側では、IBC は最終的なデータ配信のみを必要とし、認証、同期、順序付けの特性は想定しません（これらの特性は後ほど正しく定義されます）。

### Protocol relations

```
+------------------------------+                           +------------------------------+
| Distributed Ledger A         |                           | Distributed Ledger B         |
|                              |                           |                              |
| +--------------------------+ |                           | +--------------------------+ |
| | State Machine            | |                           | | State Machine            | |
| |                          | |                           | |                          | |
| | +----------+     +-----+ | |        +---------+        | | +-----+     +----------+ | |
| | | Module A | <-> | IBC | | | <----> | Relayer | <----> | | | IBC | <-> | Module B | | |
| | +----------+     +-----+ | |        +---------+        | | +-----+     +----------+ | |
| +--------------------------+ |                           | +--------------------------+ |
+------------------------------+                           +------------------------------+
```

## Operation

IBC の第一の目的は信頼性のある認証され順序付けられた通信を、独立したホスト machine 群の上で動作する module 間で行うことです。このためには以下の分野でのプロトコルロジックが必要となります。

- データのリレー（Data relay）
- データの機密性と復号しやすさ（Data confidentiality & legibility）
- 信頼性（Reliability）
- フロー制御（Flow control）
- 認証（Authentication）
- ステートフルであること（Statefulness）
- 多重化（Multiplexing）
- シリアライゼーション（Serialisation）

以下の段落では、各分野について IBC 内のプロトコルロジックの概要を説明します。

### Data relay

IBC アーキテクチャでは、module はネットワークインフラの上で直接相互にはメッセージを送信しません。代わりに、作成されたメッセージは「relayer プロセス」の監視によって物理的にリレーされます。IBC では下層のネットワークプロトコルスタック（TCP/IP、UDP/UP、QUIC/IP など）および物理的な相互接続のインフラにアクセスする relayer プロセスが一定数存在することを前提とします。これらの relayer プロセスは IBC プロトコルを実装した machine 群を監視し、継続的に各 machine の state をスキャンし、もう一方の machine 上でトランザクションを実行します。2つの machine 間 connection で正しく操作、進行されるために、IBC は最低1つ以上の正常かつ動作中の relayer プロセスしか必要としません。

### Data confidentiality & legibility

IBC プロトコルは、プロトコルの正しい動作に必要となる最小限のデータがアクセス可能かつ判読可能である（標準化された形式でシリアライズされている）ことのみを必要とします。state machine はそのデータを特定の relayer のみが利用できるように選択できます（詳細はこの仕様の範囲外です）。データは consensus state、client、connection、channel および packet 情報と、state 内に特定のキー/値ペアが含まれているか否かの proof を構築するために必要な補助的な state 構造で構成されます。別の machine に証明する必要がある全データは同じく判読可能でなければなりません。つまり、この仕様で定義される形式でシリアライズされる必要があります。

### Reliability

ネットワーク層と relayer プロセスは packet を紛失したり、並べ替えたり、複製したり、故意に無効なトランザクションを送信しようとしたり、その他ビザンチン的な振る舞いをしたりと自由に振る舞う可能性があります。こうした振る舞いによって IBC の safety や liveness が損なわれてはいけません。対策として（送信時に）IBC connection を介した各 packet にシーケンス番号を割り当てられます。これは、受信 machine 上のIBC handler（IBC プロトコルを実装する state machine の一部）によってチェックされます。また、送信 machine がさらなる packet 送信やアクションの前に、受信 machine が実際に packet を受信して処理した事実を確認できる方法が提供されます。datagram の偽造を防ぐために暗号 commitment が使用されます。送信側の machine が送信 packet にコミットし、受信側の machine がこれらのコミットをチェックするため、relayer によってリレー中に変更された datagram は拒否されます。IBC は順序付けされない channel もサポートしますが、この場合 packet の受信と送信の相対的な順序付けを強制しないものの、正確に一度だけ（exactly-once）の配信は強制されます。

### Flow control

IBC は、計算レベルまたは経済レベルでフロー制御を行うための個別の規定を提供しません。土台となる machine は計算スループットの制限と、独自のフロー制御メカニズムを持つことになるでしょう(例えば「ガス」市場など)。アプリケーションレベルの経済的なフロー制御 — 内容に応じて特定の packet のレートを制限する — は、安全性を担保し（単一の machine 上での価値を制限する）ビザンチン故障からのダメージを封じ込めるのに役立つかもしれません（データの不一致を証明するためにチャレンジ期間を許可し、その後、connection を閉じる）。例えば IBC channel 上で価値を転送するアプリケーションは、潜在的なビザンチン的振る舞いによるダメージを軽減するために block 毎の価値転送レートを制限したいと思うかもしれません。IBC は packet を拒否する機能を module に提供し、詳細は上位レベルのアプリケーションプロトコルに委ねます。

### Authentication

IBC におけるすべての datagram は認証されます。送信側 machine の consensus アルゴリズムによって finality を得た block は、暗号 commitment を介して送信 packet にコミットしなければならず、受信 側 chain の IBC handler は datagram を処理する前に datagram が送信されたことを、consensus 転写物と暗号 commitment proof の両方によって検証しなければならない。

### Statefulness

これまでに説明された信頼性やフロー制御、認証はデータストリームごとに IBC が初期化され状態情報がメンテナンスされることを必要とします。この情報は2つの抽象化、connection と channel に分かれています。各 connection オブジェクトは接続された machine の consensus state に関する情報を含みます。各 channel は moduleのペアに固有で、ネゴシエートによって定まったエンコーディングや多重化オプション、state やシーケンス番号に関する情報を含んでいます。2つの module が通信しようとした場合、machine 間にある既存の connection と channel を用いるか、まだ存在していなければそれらを初期化する必要があります。connection と channel の初期化には、複数ステップでのハンドシェイクが必要です。一度完了すればconnection では、意図された2台の machine のみが接続されていることが保証されます。channel では、2つの module が接続され将来の datagram が認証、符号化、意図通りの順序付けによってリレーされることを保証します。

### Multiplexing

1 台のホスト machine 内の多数の module が同時に IBC connection を使用できるようにするため、IBC は各 connection 内に channel のセットを提供します。各 channel は packet を順番に（順序付き module の場合）送信できるデータストリームを一意に識別し、受信側 machine の宛先 module には packet が常に一度だけ正確に送信されます。channel は通常各 machine 上の単一の module に関連付けられますが、1対多や多対1の channel も可能です。channel 数に制限はなく、土台となる machine のスループットによってのみ制限される並行スループットを容易にします。machine が consensus 情報を追跡するために必要な connection は1つだけです（したがって、consensus 転写物の検証コストは connection を使用するすべての channel で償却されます）。

### Serialisation

IBCは、そうでなければ相互に理解できない machine 間のインタフェース境界として機能し、プロトコルを正しく実装している2つの machine がお互いに理解できるため最低限必要となるデータ構造のエンコーディングと datagram フォーマットを提供しなければなりません。この目的のため、IBC 仕様では IBC を介して通信する2台の machine 間の proof でシリアライズ、リレー、チェックされる必要があるデータ構造の正準的なエンコーディングを定義します。proto3 フォーマットでこのリポジトリが提供します。

> 正準的なエンコーディング(同じ構造体が常に同じバイトにシリアライズされる)を提供する proto3 のサブセットを使用しる必要があることに注意してください。したがって、マップや未知のフィールドは禁止されます。

## Dataflow

IBC はデータが上から下（IBC packet を送信するとき）と下から上（IBC <br> packet を受信するとき）に流れる層状のプロトコルスタックとして概念化することができます。

「handler」とは、IBCプロトコルを実装する state machine の一部で、module からの呼び出しを packet に変換し、channel と connection への呼び出しを適切にルーティングする役割を担っています。

2つの chain 間の IBC packet の経路を考えてみましょう。chain はそれぞれ *A* と *B* とします。

### Diagram

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

### Steps

1. chain *A* 上
    1. Module（アプリケーション固有）
    2. Handler（複数の ICS 間で部分的に定義されます）
    3. Packet（[ICS 4](../spec/ics-004-channel-and-packet-semantics) で定義されます）
    4. Channel（[ICS 4](../spec/ics-004-channel-and-packet-semantics) で定義されます）
    5. Connection（[ICS 3](../spec/ics-003-connection-semantics) で定義されます）
    6. Client（[ICS 2](../spec/ics-002-client-semantics) で定義されます）
    7. Consensus（送信 packet を含むトランザクションを確定します）
2. オフチェーン
    1. Relayer（[ICS 18](../spec/ics-018-relayer-algorithms) で定義されます）
3. chain *B* 上
    1. Consensus（受信 packet を含むトランザクションを確定します）
    2. Client（[ICS 2](../spec/ics-002-client-semantics) で定義されます）
    3. Connection（[ICS 3](../spec/ics-003-connection-semantics) で定義されます）
    4. Channel（[ICS 4](../spec/ics-004-channel-and-packet-semantics) で定義されます）
    5. Packet（[ICS 4](../spec/ics-004-channel-and-packet-semantics) で定義されます）
    6. Handler（複数の ICS 間で部分的に定義されます）
    7. Module（アプリケーション固有）
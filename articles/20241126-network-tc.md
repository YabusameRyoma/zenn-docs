---
title: "tcコマンドでネットワークの障害を再現して動作検証する方法"
emoji: "⛓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tc", "network", "debug", "test"]
published: true
---

ネットワーク障害を再現してアプリケーションの耐障害性や挙動を確認することは、信頼性の高いシステムを構築するために欠かせません。本記事では、Linuxの`tc`（Traffic Control）コマンドを使用して、ネットワーク遅延、パケットロス、帯域制限などの障害をシミュレーションする方法を詳しく解説します。また、柔軟なシェルスクリプトを活用して、効率的に設定・リセットを行うTipsも紹介します。

## **1. tcコマンドとは？**

`tc`コマンドは、Linuxでネットワークトラフィックを制御するためのツールです。遅延やパケットロス、帯域制限、さらにはパケットの破損や複製など、さまざまなネットワーク障害を再現できます。

### **主な用途**

- アプリケーションの耐障害性テスト
- システムやサービスのデバッグ
- 特定のネットワーク条件での動作確認

## **2. tcコマンドの事前準備**

### **必要な環境**

1. **Linux OS**:
   - Ubuntu, Debian, AlmaLinux, Rocky Linux など

2. **`iproute2`パッケージ**:

   - `tc`コマンドを含むパッケージ
   - インストールコマンド例：

     #### **Ubuntu/Debian 系の場合**:

     ```bash
     sudo apt update
     sudo apt install iproute2
     ```

     #### **AlmaLinux/Rocky Linux（RHEL系）の場合**:

     ```bash
     sudo dnf install iproute
     ```

3. **管理者権限**:

   - `tc`コマンドはroot権限が必要です

## **3. 基本的なコマンド例**

以下に、よく使用する`tc`コマンドを示します。これらの設定は、ネットワークインターフェース`eth0`に適用されます。

### **3.1 遅延を追加**

```bash
tc qdisc add dev eth0 root netem delay 100ms
```

- **説明**: インターフェース`eth0`に100msの固定遅延を追加します。

#### **オプションの詳細**

- **`delay 100ms`**:
  - トラフィックに一定の遅延（100ミリ秒）を追加します。
  - **主な用途**:
    - 地理的に離れたサーバーとの通信を模擬する
    - 通信速度が遅いネットワーク環境のテスト

- **`delay 100ms 20ms distribution normal`**:
  - 遅延に揺らぎ（ジッタ）を追加します。
  - `20ms`はジッタの範囲（標準偏差）を表します。
  - **主な用途**:
    - モバイルネットワークやWi-Fi環境など、不安定な遅延を再現

### **3.2 パケットロスを追加**

```bash
tc qdisc add dev eth0 root netem loss 10%
```

- **説明**: 全トラフィックの10%をランダムにドロップします。

#### **オプションの詳細**

- **`loss 10%`**:
  - トラフィックの10%をランダムに削除
  - **主な用途**:
    - 不安定な通信環境をシミュレーション（例: 地方のモバイル回線）

- **`loss 10% 25%`**:
  - 二値モデルでパケットロスを設定します。基本ロス率が10%、さらに25%の確率で追加のパケットがロスします。
  - **主な用途**:
    - 極端に不安定なネットワークを再現

### **3.3 帯域制限**

帯域幅を制限するには、`tbf`（Token Bucket Filter）を使用します。この例では、帯域幅を1Mbpsに制限し、一定のバーストと遅延を設定します。

```bash
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms
```

#### **各オプションの説明**

- **`rate 1mbit`**:
  - 最大帯域幅を1Mbpsに制限します。
  - `kbit`（キロビット/秒）や`bit`（ビット/秒）の単位も使用可能。
  - **主な用途**:
    - 制限されたネットワーク環境（低速回線や共有ネットワーク）を模擬

- **`burst 32kbit`**:
  - 短期間のトラフィック急増（バースト）を許容する最大データ量。
  - 値が小さいと、トラフィックスパイクが抑制されますが、低遅延が求められるシナリオではデータドロップの可能性が高まります。
  - **主な用途**:
    - スパイク耐性をテストする

- **`latency 400ms`**:
  - バッファリングされたトラフィックが処理されるまでの最大遅延時間。
  - 指定した時間内に送信できないトラフィックはドロップされます。
  - **主な用途**:
    - 長時間のキューイングが発生する状況を模擬する

### **3.4 パケット破損を追加**

```bash
tc qdisc add dev eth0 root netem corrupt 5%
```

- **説明**: トラフィックの5%を意図的に破損させます。

#### **オプションの詳細**

- **`corrupt 5%`**:
  - 全トラフィックの5%が破損します。
  - **主な用途**:
    - アプリケーションが破損したデータをどのように処理するかを検証する
    - ファイル転送やストリーミングの耐性テスト

### **3.5 パケット複製を追加**

```bash
tc qdisc add dev eth0 root netem duplicate 2%
```

- **説明**: トラフィックの2%を複製します。

#### **オプションの詳細**

- **`duplicate 2%`**:
  - 全トラフィックの2%が重複して送信されます。
  - **主な用途**:
    - 複製データによるトラフィック増加への耐性をテストする
    - ループやデータストームを再現

### **3.6 パケット順序を入れ替え**

```bash
tc qdisc add dev eth0 root netem reorder 25% 50%
```

- **説明**: トラフィックの25%をランダムに順序入れ替え、50%のトラフィックは保護（順序保持）されます。

#### **オプションの詳細**

- **`reorder 25%`**:
  - パケットの順序をランダムに入れ替える確率。
  - **主な用途**:
    - 順序が重要なデータストリーム（例: ビデオ通話）の影響を検証する

- **`reorder 25% 50%`**:
  - 25%のパケットを入れ替え、50%のトラフィックを順序保持します。
  - **主な用途**:
    - 不安定な順序性を伴うネットワーク状況を再現

### **3.7 複数の設定を同時に適用**

```bash
tc qdisc add dev eth0 root netem delay 100ms loss 5% corrupt 1% reorder 10% 30%
```

- **説明**: 以下の設定を同時に適用：
  - 遅延100ms
  - パケットロス5%
  - パケット破損1%
  - 順序入れ替え10%、保護30%
- **主な用途**:
  - 複雑なネットワーク障害を再現する

### **3.8 帯域制限と遅延を同時に適用**

`netem`と`tbf`を組み合わせて、遅延やパケットロスと同時に帯域制限を行うには、階層的なキューイングディシプリン（qdisc）を使用します。以下は、`htb`を使用して階層化する例です。

```bash
# 既存のqdiscを削除
tc qdisc del dev eth0 root

# ルートにhtbを設定
tc qdisc add dev eth0 root handle 1: htb default 12

# クラスを追加（帯域制限）
tc class add dev eth0 parent 1: classid 1:1 htb rate 1mbit ceil 1mbit

# netemをクラスに適用（遅延やパケットロス）
tc qdisc add dev eth0 parent 1:1 handle 10: netem delay 100ms loss 5%
```

- **説明**:
  - **htb**（Hierarchical Token Bucket）をルートqdiscとして設定し、その下にクラスを作成
  - クラスに対して帯域制限を設定
  - `netem`をクラスに適用して、遅延やパケットロスをシミュレーション

## **4. 設定のリセット方法**

設定をリセットするには以下のコマンドを使用します。

```bash
tc qdisc del dev eth0 root
```

- **説明**: `eth0`のすべての設定を削除します。

## **5. シェルスクリプトで効率化**

`tc`コマンドを効率的に利用するためのシェルスクリプトを以下に示します。このスクリプトは、ネットワークインターフェース名を指定し、柔軟にパラメーターを設定できるようになっています。

### **スクリプト**

```bash
#!/bin/bash

IFACE=$1
ACTION=$2

if [[ -z $IFACE ]]; then
    echo "エラー: インターフェース名が指定されていません。"
    echo "使用方法: $0 <インターフェース名> {apply|reset} [delay=遅延] [loss=パケットロス] [rate=帯域幅] [corrupt=破損率] [duplicate=複製率] [reorder=入れ替え率]"
    exit 1
fi

DELAY="0ms"
LOSS="0%"
RATE=""
CORRUPT="0%"
DUPLICATE="0%"
REORDER="0%"

for ARG in "${@:3}"; do
    case $ARG in
        delay=*) DELAY="${ARG#*=}" ;;
        loss=*) LOSS="${ARG#*=}" ;;
        rate=*) RATE="${ARG#*=}" ;;
        corrupt=*) CORRUPT="${ARG#*=}" ;;
        duplicate=*) DUPLICATE="${ARG#*=}" ;;
        reorder=*) REORDER="${ARG#*=}" ;;
    esac
done

apply_tc() {
    echo "ネットワーク設定を適用中... インターフェース: $IFACE"
    # 既存のqdiscを削除
    tc qdisc del dev $IFACE root 2> /dev/null

    # ルートにhtbを設定
    tc qdisc add dev $IFACE root handle 1: htb default 12

    # クラスを追加（帯域制限）
    if [[ -n $RATE ]]; then
        tc class add dev $IFACE parent 1: classid 1:1 htb rate $RATE ceil $RATE
    else
        tc class add dev $IFACE parent 1: classid 1:1 htb rate 1000mbit ceil 1000mbit
    fi

    # netemをクラスに適用
    tc qdisc add dev $IFACE parent 1:1 handle 10: netem delay $DELAY loss $LOSS corrupt $CORRUPT duplicate $DUPLICATE reorder $REORDER

    echo "適用しました: 遅延=$DELAY, パケットロス=$LOSS, 帯域=${RATE:-"制限なし"}, 破損=$CORRUPT, 複製=$DUPLICATE, 入れ替え=$REORDER, 対象インターフェース=$IFACE"
}

reset_tc() {
    echo "ネットワーク設定をリセット中... インターフェース: $IFACE"
    tc qdisc del dev $IFACE root
    echo "$IFACE のネットワーク設定をリセットしました"
}

case $ACTION in
    apply) apply_tc ;;
    reset) reset_tc ;;
    *) 
        echo "使用方法: $0 <インターフェース名> {apply|reset} [delay=遅延] [loss=パケットロス] [rate=帯域幅] [corrupt=破損率] [duplicate=複製率] [reorder=入れ替え率]"
        exit 1
        ;;
esac
```

### **スクリプトの使い方**

1. **ネットワーク設定を適用**:

```bash
./network_tc.sh eth0 apply delay=100ms loss=2% corrupt=1% rate=1mbit
```

2. **設定をリセット**:

```bash
./network_tc.sh eth0 reset
```

- **注意点**:
  - `rate` パラメータを使用する場合、`htb` を利用して帯域制限を適用しています。
  - スクリプトは既存のqdiscを削除してから新しい設定を適用します。

## **6. まとめ**

`tc`コマンドは、ネットワーク障害をシミュレーションするための非常に強力なツールです。本記事で紹介した方法やスクリプトを活用することで、効率的に設定やリセットが行えます。特に、`netem`と`htb`を組み合わせることで、遅延やパケットロスと同時に帯域制限も適用できます。ぜひ、開発やテスト環境で活用してみてください。

---

## **7. 参考資料**

- [Linux Traffic Control ドキュメント](https://man7.org/linux/man-pages/man8/tc.8.html)

---

## おまけ：tcコマンドのマニュアル日本語訳

### 名前  

**tc** - トラフィック制御設定の表示/操作  

### 構文  
```  
tc [ オプション ] qdisc [ add | change | replace | link | delete ]  
    dev デバイス名 [ parent qdisc-id | root ] [ handle qdisc-id ]  
    [ ingress_block ブロックインデックス ] [ egress_block ブロックインデックス ] qdisc  
    [ qdisc 特定のパラメータ ]

tc [ オプション ] class [ add | change | replace | delete | show ]  
    dev デバイス名 parent qdisc-id [ classid class-id ]  
    qdisc [ qdisc 特定のパラメータ ]  

tc [ オプション ] filter [ add | change | replace | delete | get ]  
    dev デバイス名 [ parent qdisc-id | root ] [ handle filter-id ] protocol  
    プロトコル prio 優先度 filtertype [ filtertype 特定のパラメータ ]  
    flowid フローID

tc [ オプション ] chain [ add | delete | get ] dev デバイス名  
    [ parent qdisc-id | root ] filtertype [ filtertype 特定のパラメータ ]  
```

---

### 説明  
`tc` は Linux カーネルのトラフィック制御 (Traffic Control) を設定するためのコマンドです。トラフィック制御は以下の要素から成ります：

#### 1. **整形 (Shaping)**  
トラフィックの送信速度を制御します。帯域幅を制限するだけでなく、ネットワークの動作を改善するためにバーストを平滑化することも含まれます。整形は送信時 (egress) に行われます。

#### 2. **スケジューリング (Scheduling)**  
パケットの送信スケジュールを調整することで、リアルタイム通信などの応答性を向上させつつ、大量データ転送にも帯域を確保します。パケットの並び替え（優先度付け）は送信時のみ行われます。

#### 3. **ポリシング (Policing)**  
整形が送信トラフィックを扱うのに対し、ポリシングは受信トラフィックを管理します。ポリシングは受信時 (ingress) に適用されます。

#### 4. **ドロップ (Dropping)**  
設定された帯域幅を超えたトラフィックは、その場でドロップされることがあります（送信時および受信時の両方）。

---

### トラフィック制御の主要オブジェクト  
トラフィックの処理は以下の3種類のオブジェクトによって制御されます：

- **qdisc（キューイングディシプリン、Queueing Discipline）**  
- **class（クラス）**  
- **filter（フィルタ）**  

---

#### QDISC（キューイングディシプリン）
qdisc は "queueing discipline" の略で、トラフィック制御の基本要素です。カーネルがネットワークインターフェイスにパケットを送信する際、それはインターフェイスに設定された qdisc にエンキューされます。その後すぐにカーネルは可能な限り多くのパケットを qdisc から取り出し、ネットワークアダプタドライバに渡します。

- **シンプルな qdisc の例**  
  `pfifo` は最もシンプルな qdisc で、特に処理を行わない単純な「先入れ先出し (FIFO)」のキューです。ただし、ネットワークインターフェイスが一時的に処理できない場合にトラフィックを一時的に蓄積します。

---

#### CLASSES（クラス）
一部の qdisc はクラスを含むことができ、さらにそのクラス内に別の qdisc を含むことが可能です。トラフィックはこれらの内部 qdisc にエンキューされます。

- **例**  
  クラス付き qdisc は特定の種類のトラフィックを優先することができます。たとえば、特定のクラスからパケットを優先的に取り出すように設定できます。

---

#### FILTERS（フィルタ）
フィルタは、クラス付き qdisc がパケットをどのクラスにエンキューするかを決定するために使用されます。サブクラスを持つクラスにトラフィックが到着すると、分類が必要になります。この際、フィルタが利用されます。

- **動作**  
  クラスにアタッチされたすべてのフィルタが呼び出され、いずれかが「判定」を返すまで処理されます。判定が得られない場合、別の基準が適用されることがあります（qdisc によって異なります）。  
  フィルタは qdisc 内に存在し、それを直接コントロールするものではありません。

- **利用可能なフィルタの種類**  
  - `basic`: ematch 式に基づいてパケットをフィルタリングします。詳細は `tc-ematch(8)` を参照。  
  - `bpf`: (e)BPF を利用してフィルタリングします。詳細は `tc-bpf(8)` を参照。  
  - `cgroup`: プロセスのコントロールグループに基づいてフィルタリングします。詳細は `tc-cgroup(8)` を参照。  
  - `flow`, `flower`: フローに基づく分類を行います。詳細は `tc-flow(8)` および `tc-flower(8)` を参照。  
  - `fw`: fwmark に基づいて分類します。詳細は `tc-fw(8)` を参照。  
  - `route`: ルーティングテーブルに基づいて分類します。詳細は `tc-route(8)` を参照。  
  - `u32`: パケットデータを汎用的にフィルタリングします。詳細は `tc-u32(8)` を参照。  
  - `matchall`: すべてのパケットにマッチするフィルタです。詳細は `tc-matchall(8)` を参照。

---

### QEVENTS（キューイングイベント）
qdisc は、特定のイベントが発生した際にユーザーが設定したアクションを実行することができます。各 qevent は未使用にすることも、ブロックを関連付けることも可能です。このブロックには `tc block BLOCK_IDX` 構文を使用してフィルタがアタッチされます。qevent に関連付けられたアクションポイントがトリガーされた際に、このブロックが実行されます。

#### **例**  
以下の設定では、特定の条件下でパケットがミラーリングされます：
```bash
tc qdisc add dev eth0 root handle 1: red limit 500K avpkt 1K \
   qevent early_drop block 10
tc filter add block 10 matchall action mirred egress mirror dev eth1
```

---

### CLASSLESS QDISCS（クラスなし qdisc）
クラスなし qdisc は以下の種類があります：

- **choke**  
  CHOKe (CHOose and Keep/CHOose and Kill) は、応答性のあるフローを保持し、応答性のないフローを抑制するための qdisc です。RED（Random Early Detection）の派生であり、設定方法は RED と類似しています。

- **codel**  
  CoDel（Controlled Delay）は、RED の欠点を克服するために開発されたアダプティブな「ノーノブ」型のキュー管理アルゴリズムです。

- **[p|b]fifo**  
  シンプルな FIFO（First In, First Out）動作を提供します。パケット単位またはバイト単位で制限されます。

- **fq**  
  TCP ペーシングを実現し、数百万の同時フローにスケール可能なフェアキューイングスケジューラです。

- **fq_codel**  
  Fair Queuing Controlled Delay は、フェアキューイングと CoDel AQM スキームを組み合わせた qdisc です。フローごとに公平な帯域幅分配を行い、各フローを CoDel キューイングディシプリンで管理します。

- **fq_pie**  
  FQ-PIE（Flow Queuing with Proportional Integral controller Enhanced）は、フェアキューイングと PIE AQM を組み合わせた qdisc です。

- **gred**  
  Generalized Random Early Detection は、複数の RED キューを組み合わせて、異なるドロップ優先度を実現します。

- **hhf**  
  Heavy-Hitter Filter は、小規模なフローと大規模なフローを区別し、重いフローを優先度の低いキューに移動させることで、重要なトラフィックの遅延を抑制します。

- **ingress**  
  入力トラフィックに適用される特殊な qdisc です。トラフィックをフィルタリングし、ポリシングします。

- **mqprio**  
  Multiqueue Priority Qdisc は、優先度を基にトラフィックフローをハードウェアキューにマッピングします。

- **multiq**  
  複数の送信キューを持つデバイス向けに最適化された qdisc です。

- **netem**  
  ネットワークエミュレータ。遅延、パケット損失、重複などのネットワーク特性を追加できます。

- **pfifo_fast**  
  高度なルータ向けカーネルの標準 qdisc です。Type of Service フラグやパケットの優先度を考慮する3バンドキューで構成されています。

- **pie**  
  Proportional Integral controller Enhanced は、遅延を制御する理論に基づくアクティブキュー管理スキームです。

- **red**  
  Random Early Detection は、設定された帯域幅に近づくとランダムにパケットをドロップすることで、物理的な輻輳をシミュレーションします。

- **sfb**  
  Stochastic Fair Blue は、パケット損失やリンク利用率に基づいて輻輳を管理します。

- **sfq**  
  Stochastic Fairness Queueing は、各「セッション」に対して順番にパケットを送信する公平性を確保するキューイング方式です。

- **tbf**  
  Token Bucket Filter は、トラフィックを正確に設定された速度まで減速させるのに適しています。

---

### クラスなし qdisc の設定 (CONFIGURING CLASSLESS QDISCS)

クラスなし qdisc はクラスをサポートしないため、デバイスのルートにのみアタッチできます。構文は以下の通りです：

#### **追加**
```bash
tc qdisc add dev デバイス名 root QDISC QDISC-パラメータ
```

#### **削除**
```bash
tc qdisc del dev デバイス名 root
```

デフォルトでは、qdisc を設定しない場合に `pfifo_fast` が自動的に割り当てられます。

---

### クラス付き qdisc (CLASSFUL QDISCS)

クラス付き qdisc の種類は以下の通りです：

- **ATM**  
  フローを基礎となる非同期転送モード (ATM) デバイスの仮想回線にマップします。

- **DRR (Deficit Round Robin)**  
  ストキャスティック・フェアネス・キューイング (SFQ) の柔軟な代替です。RED や他の qdisc を特定のトラフィックに適用する場合に役立ちます。フィルタを使用してパケットを分類する必要があります。

- **ETS (Enhanced Transmission Selection)**  
  PRIO qdisc と DRR qdisc の機能を統合し、厳格なバンドと帯域幅共有バンドを簡単に構成できるようにします。

- **HFSC (Hierarchical Fair Service Curve)**  
  リーフクラスに対して正確な帯域幅と遅延の割り当てを保証します。インタラクティブなセッションに適した低遅延を実現するため、パケットのドロップを活用します。

- **HTB (Hierarchy Token Bucket)**  
  階層型のリンク共有クラスを実装します。帯域幅の保証とクラス間の共有上限の設定が可能です。

- **PRIO**  
  優先度付けを容易にするコンテナ型 qdisc です。優先度の高いクラスが空でない限り、低いクラスの送信はできません。Type of Service ビットに基づくデフォルトの分類ルールがあります。

- **QFQ (Quick Fair Queueing)**  
  O(1) スケジューラで、グループ数やパケット長に関係なく一定コストで動作します。

---

### 動作の理論 (THEORY OF OPERATION)

#### **クラスの階層構造**
クラスはツリー構造を形成し、各クラスは1つの親クラスを持ちます。クラスは複数の子クラスを持つことが可能です。

- **動的クラス追加**  
  一部の qdisc（例: HTB）はランタイムでクラスを追加できますが、他の qdisc（例: PRIO）は静的な数の子クラスを持ちます。

- **葉 (リーフ) qdisc**  
  各クラスはデフォルトで `pfifo` の動作を持つ葉 qdisc を含みます。ただし、別の qdisc をアタッチすることも可能です。

- **分類 (Classification)**  
  パケットがクラス付き qdisc に入ると、以下の3つの基準に基づいてサブクラスに分類されます：
  1. **tc フィルタ**  
     パケットヘッダのすべてのフィールドや iptables によるファイアウォールマークに基づいて分類します。
  2. **サービス種別 (Type of Service)**  
     TOS フィールドに基づく組み込みルールを利用します。
  3. **skb->priority**  
     ユーザー空間プログラムが SO_PRIORITY オプションを使用して `skb->priority` フィールドにクラス ID をエンコードします。

---

### 命名規則 (NAMING)

すべての qdisc、クラス、フィルタには ID があり、指定するか自動的に割り当てられます。

- **ID の構成**  
  ID は「メジャー番号:マイナー番号」という形式で表されます（例: `10:20`）。メジャー番号は qdisc を指し、マイナー番号はクラスを指します。

- **特別な値**  
  - `root`: メジャーとマイナーがすべて1の状態を指します（例: `ffff:ffff`）。
  - `unspecified`: メジャーとマイナーがすべて0の状態を指します（例: `0:0`）。

---

### パラメータ (PARAMETERS)

以下のパラメータは広く使用されます。個別の qdisc の詳細については、それぞれの man ページを参照してください。

#### **帯域幅やレート (RATES)**
- 数値には単位（SI/IEC）の指定が可能です。例: `5mbit`, `10kbps`
- 利用可能な単位：
  - `kbit`, `mbit`, `gbit`, `bps`（SI単位）
  - `kib`, `mib`, `gib`, `tib`（IEC単位）

#### **時間 (TIMES)**
- 秒単位: `1s`, `0.5sec`
- ミリ秒単位: `100ms`
- マイクロ秒単位: `1000us`

#### **データ量 (SIZES)**
- バイト単位: `128b`
- キロバイト単位: `64k`
- メガバイト単位: `1mb`

---

### TC コマンド (TC COMMANDS)

`tc` コマンドは、qdisc、クラス、フィルタに対して以下の操作を実行できます。

#### **1. 追加 (add)**
- qdisc、クラス、フィルタをノードに追加します。
- すべてのエンティティには親が必要です。親は ID を指定するか、デバイスのルートに直接アタッチします。
- 作成する qdisc またはフィルタには `handle` パラメータで名前を付けられます。クラスには `classid` パラメータを使用します。

**例:**
```bash
tc qdisc add dev eth0 root handle 1: pfifo
```

#### **2. 削除 (delete)**
- qdisc を削除するには、その `handle` を指定します（`root` も可）。
- 削除すると、すべてのサブクラスや関連するフィルタも自動的に削除されます。

**例:**
```bash
tc qdisc del dev eth0 root
```

#### **3. 変更 (change)**
- 既存のエンティティを「その場で」変更します。
- `handle` や `parent` は変更できません。ノードを移動させることはできません。

**例:**
```bash
tc qdisc change dev eth0 root handle 1: fq_codel
```

#### **4. 置き換え (replace)**
- 既存のノード ID に対して、ほぼアトミックに削除/追加を行います。
- ノードがまだ存在しない場合は作成されます。

**例:**
```bash
tc qdisc replace dev eth0 root fq_codel
```

#### **5. 取得 (get)**
- デバイス (`DEV`)、qdisc ID、優先度 (`priority`)、プロトコル (`protocol`)、フィルタ ID を指定して、単一のフィルタを表示します。

**例:**
```bash
tc filter get dev eth0 protocol ip prio 1
```

#### **6. 表示 (show)**
- 指定したインターフェースにアタッチされたすべてのフィルタを表示します。
- 有効な親 ID を指定する必要があります。

**例:**
```bash
tc filter show dev eth0
```

#### **7. リンク (link)**
- qdisc のみに使用可能で、既存ノードに対して置き換え操作を実行します。

---

### モニタリング (MONITOR)

`tc` ユーティリティは、カーネルによって生成されたイベント（qdisc、フィルタ、アクションの追加/削除、変更など）をモニタリングすることが可能です。

#### **コマンド例**
- **file:**  
  指定されたファイルを開き、その内容をダンプします。ファイルはバイナリ形式で、netlink メッセージを含む必要があります。

**例:**
```bash
tc monitor
```

---

### オプション (OPTIONS)

`tc` コマンドで使用できる主なオプションは以下の通りです。

#### **1. バッチ処理**
- **`-b, -batch`**  
  指定したファイルまたは標準入力からコマンドを読み取り、実行します。最初のエラーで終了します。

- **`-force`**  
  バッチモードでエラーが発生しても `tc` を終了させません。ただし、エラーが発生した場合、アプリケーションの終了コードは非ゼロになります。

**例:**
```bash
tc -b commands.txt
```

#### **2. 出力フォーマット**
- **`-o, -oneline`**  
  各レコードを1行で出力します。行の改行を `\` に置き換えます。

- **`-N, -Numeric`**  
  プロトコル、スコープ、dsfield などを人間が読める形式ではなく、数値として直接表示します。

- **`-p, -pretty`**  
  読みやすくフォーマットされた出力を提供します。

- **`-j, -json`**  
  JSON 形式で結果を表示します。

**例:**
```bash
tc -s qdisc show dev eth0 -json
```

---

### フォーマットオプション (FORMAT)

`show` コマンドには以下のフォーマットオプションを追加できます：

- **`-s, -stats, -statistics`**  
  パケット使用量に関する統計情報を追加で出力します。

- **`-g, -graph`**  
  クラスを ASCII グラフとして表示します。

**例:**
```bash
tc -g -s class show dev eth0
```

---

### 例 (EXAMPLES)

以下は `tc` コマンドの実際の使用例です。

#### **1. クラスを ASCII グラフで表示**
```bash
tc -g class show dev eth0
```
このコマンドは、`eth0` インターフェース上のクラスを ASCII グラフとして表示します。

#### **2. 統計情報付きでクラスをグラフ表示**
```bash
tc -g -s class show dev eth0
```
このコマンドは、`eth0` インターフェース上のクラスを統計情報付きでグラフ表示します。

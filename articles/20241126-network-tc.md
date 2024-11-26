---
title: "tcコマンドでネットワークの障害を再現して動作検証するTips"
emoji: "⛓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tc", "network", "debug", "test"]
published: false
---

ネットワーク障害を再現してアプリケーションの耐障害性や挙動を確認することは、信頼性の高いシステムを構築するために欠かせません。本記事では、Linuxの`tc`（Traffic Control）コマンドを使用して、ネットワーク遅延、パケットロス、帯域制限などの障害をシミュレーションする方法を詳しく解説します。また、柔軟なシェルスクリプトを活用して、効率的に設定・リセットを行うTipsも紹介します。

## **1. tcコマンドとは？**

`tc`コマンドは、Linuxでネットワークトラフィックを制御するためのツールです。遅延やパケットロス、帯域制限、さらにはパケットの破損や複製など、さまざまなネットワーク障害を再現できます。

### **主な用途**

- アプリケーションの耐障害性テスト。
- システムやサービスのデバッグ。
- 特定のネットワーク条件での動作確認。

## **2. tcコマンドの事前準備**

### **必要な環境**

1. **Linux OS**:
   - Ubuntu, Debian, AlmaLinux, Rocky Linux など。

2. **`iproute2`パッケージ**:

   - `tc`コマンドを含むパッケージ。
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

   - `tc`コマンドはroot権限が必要です。

## **3. 基本的なコマンド例**

以下に、よく使用する`tc`コマンドを示します。これらの設定は、ネットワークインターフェース`eth0`に適用されます。

### **3.1 遅延を追加**

```bash
tc qdisc add dev eth0 root netem delay 100ms
```
- **説明**: 遅延100msをインターフェース`eth0`に追加します。

### **3.2 パケットロスを追加**

```bash
tc qdisc change dev eth0 root netem loss 10%
```
- **説明**: 全トラフィックの10%をランダムにドロップします。

### **3.3 帯域制限**

帯域幅を制限するには、`tbf`（Token Bucket Filter）を使用します。この例では、帯域幅を1Mbpsに制限し、一定のバーストと遅延を設定します。

```bash
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms
```

#### **各オプションの説明**

- **`rate 1mbit`**:
  - 帯域幅を1Mbps（1メガビット/秒）に制限します。
  - 他にも`kbps`や`bps`の単位が使用できます。

- **`burst 32kbit`**:
  - 一時的に許容される最大トラフィック量を指定します。
  - データが急増した際、指定されたサイズ（32kbit）までバッファリングされ、制限された帯域幅内で送信されます。
  - 小さい値にすると、スパイクを抑えますが、低遅延が要求される場合に問題を引き起こす可能性があります。

- **`latency 400ms`**:
  - バッファリングが維持される最大遅延時間を指定します。
  - 指定した時間内に処理できないトラフィックはドロップされます。
  - 高すぎる値を指定すると、遅延が大きくなりすぎてアプリケーションに影響を与える可能性があります。

### **3.4 パケット破損を追加**

```bash
tc qdisc add dev eth0 root netem corrupt 5%
```
- **説明**: トラフィックの5%を破損させます。

### **3.5 パケット複製を追加**

```bash
tc qdisc add dev eth0 root netem duplicate 2%
```
- **説明**: トラフィックの2%を複製します。

### **3.6 パケット順序を入れ替え**

```bash
tc qdisc add dev eth0 root netem reorder 25% 50%
```
- **説明**: トラフィックの25%を順序入れ替え、50%を保護します。

### **3.7 複数の設定を同時に適用**

```bash
tc qdisc add dev eth0 root netem delay 100ms loss 5% corrupt 1% reorder 10% 30%
```
- **説明**: 以下を同時に適用：
  - 遅延100ms
  - パケットロス5%
  - パケット破損1%
  - 順序入れ替え10%、保護30%

## **4. 設定のリセット方法**

設定をリセットするには以下のコマンドを使用します。

```bash
tc qdisc del dev eth0 root
```
- **説明**: `eth0`のすべての設定を削除します。

## **5. シェルスクリプトで効率化**

`tc`コマンドを効率的に利用するためのシェルスクリプトを以下に示します。このスクリプトは、ネットワークインターフェース名を指定し、柔軟にパラメーターを設定できるようになっています。

### **スクリプト例**

```shell
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
RATE="unlimited"
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
    tc qdisc add dev $IFACE root netem delay $DELAY loss $LOSS rate $RATE corrupt $CORRUPT duplicate $DUPLICATE reorder $REORDER
    echo "適用しました: 遅延=$DELAY, パケットロス=$LOSS, 帯域=$RATE, 破損=$CORRUPT, 複製=$DUPLICATE, 入れ替え=$REORDER, 対象インターフェース=$IFACE"
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

### スクリプトの使い方

1. ネットワーク設定を適用:

```bash
./network_tc.sh eth0 apply delay=100ms loss=2% corrupt=1%
```

2. 設定をリセット:

```bash
./network_tc.sh eth0 reset
```

## **6. まとめ**

`tc`コマンドは、ネットワーク障害をシミュレーションするための非常に強力なツールです。本記事で紹介したスクリプトを活用することで、効率的に設定やリセットが行えます。ぜひ、開発やテスト環境で活用してみてください。

## **7. 参考資料**

- [Linux Traffic Control ドキュメント](https://man7.org/linux/man-pages/man8/tc.8.html)

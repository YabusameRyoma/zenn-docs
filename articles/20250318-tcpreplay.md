---
title: "ネットワーク通信を記録・変更・再生する方法(tcpdump,tcprewrite,tcpreplay)"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ネットワーク","デバッグ","セキュリティ","パケットキャプチャ","通信解析"]
published: true
---
# 1. 背景と目的

ネットワークの動作を調べたり、セキュリティの確認をしたりするときに、通信データを記録（キャプチャ）し、あとで再現できると便利です。これにより、問題の原因を特定したり、テストを行ったりすることができます。

この記事では、以下の3つのツールを使った手順を紹介します。

- **tcpdump**  
  ネットワーク上の通信データを記録し、ファイルとして保存するツール。特定の種類の通信だけを選んで記録することもできます。

- **tcprewrite**  
  記録した通信データの一部（送信元や宛先のアドレスなど）を書き換えるツール。これを使えば、データを別の環境でも再現しやすくなります。

- **tcpreplay**  
  書き換えた通信データを、実際のネットワーク上に再送信するツール。繰り返し再生することもできるため、継続的なテストが可能になります。

---

# 2. ツールのインストール方法

## Linux（Ubuntu/Debian系）

- **tcpdump**
  ```bash
  sudo apt-get update
  sudo apt-get install tcpdump
  ```

- **tcpreplay（tcprewriteを含む）**
  ```bash
  sudo apt-get install tcpreplay
  ```

## Red Hat系（CentOSなど）

- **tcpdump**
  ```bash
  sudo yum install tcpdump
  # または
  sudo dnf install tcpdump
  ```

- **tcpreplay**
  ```bash
  sudo yum install tcpreplay
  # または
  sudo dnf install tcpreplay
  ```

## macOS（Homebrewを使用）

- **tcpdump**（macOSには標準で含まれていますが、最新バージョンを使いたい場合）
  ```bash
  brew install tcpdump
  ```

- **tcpreplay**
  ```bash
  brew install tcpreplay
  ```

---

# 3. 通信データの記録（tcpdump）

tcpdumpを使うと、ネットワーク上の通信データを記録できます。必要な情報だけを記録するように、特定の条件（通信相手やポート番号など）を設定することもできます。

## 3.1 基本的な記録コマンド

特定のコンピュータとの通信を記録する場合、以下のように実行します。

```bash
sudo tcpdump -i eth0 host 192.168.1.100 -w capture.pcap
```

## 3.2 通信の種類（プロトコル）で記録を絞り込む

- **TCP（一般的な通信）だけを記録**
  ```bash
  sudo tcpdump -i eth0 tcp -w tcp_capture.pcap
  ```

- **UDP（動画や音声などの通信）だけを記録**
  ```bash
  sudo tcpdump -i eth0 udp -w udp_capture.pcap
  ```

## 3.3 ポート番号で記録を絞り込む

- **特定のポート番号（例：3000番）を使う通信だけを記録**
  ```bash
  sudo tcpdump -i eth0 port 3000 -w capture.pcap
  ```

## 3.4 複数の条件を組み合わせる

特定のコンピュータとの通信のうち、UDPでポート3000を使うデータだけを記録する場合は、次のように指定します。

```bash
sudo tcpdump -i eth0 host 192.168.1.100 and udp port 3000 -w filtered_capture.pcap
```

---

# 4. 通信データの書き換え（tcprewrite）

記録したデータを、そのまま別の環境で使うのが難しいこともあります。たとえば、記録したデータが「192.168.1.100」というコンピュータとの通信だった場合、別のネットワークでは同じIPアドレスを使えないかもしれません。

このような場合、tcprewriteを使ってIPアドレスを書き換えることができます。

## 4.1 宛先IPアドレスの変更

以下のコマンドを実行すると、記録したデータの中で「192.168.1.100」宛ての通信を、「10.0.0.100」宛てに変更できます。

```bash
tcprewrite --infile=capture.pcap --outfile=rewritten.pcap --dstipmap=192.168.1.100:10.0.0.100
```

**ポイント:**
- `--infile` で元のデータを指定
- `--outfile` で変更後のデータを指定
- `--dstipmap` で、変更する宛先IPアドレスを指定

また、MACアドレス（ネットワーク機器の識別番号）やポート番号も書き換えることができます。

---

# 5. 通信データの再生（tcpreplay）

記録したデータをネットワーク上に再送信することで、特定の動作を再現したり、テストしたりできます。

## 5.1 通常の再生

ネットワークインターフェース `eth0` に記録データを流す場合、次のコマンドを実行します。

```bash
sudo tcpreplay -i eth0 rewritten.pcap
```

## 5.2 繰り返し再生

テストのために、同じ通信を何度も繰り返して流したい場合は、`-l` オプションを使います。

たとえば、3回繰り返して再生する場合は以下のようにします。

```bash
sudo tcpreplay -i eth0 -l 3 rewritten.pcap
```

**注意:** `-l 0` を指定すると無限ループになり、ネットワークに負荷をかける可能性があるため注意が必要です。

---

# 6. まとめと注意点

## 6.1 ワークフローのまとめ

1. **記録（キャプチャ）:**  
   tcpdumpを使って、必要な通信データを記録する。

2. **書き換え:**  
   tcprewriteを使い、記録したデータのIPアドレスなどを変更する。

3. **再生:**  
   tcpreplayを使って、書き換えたデータをネットワーク上に再送信する。

## 6.2 注意点

- **管理者権限が必要:**  
  tcpdumpやtcpreplayを実行するには管理者権限（root権限）が必要です。

- **ネットワーク環境への影響:**  
  実際のネットワークに影響を与える可能性があるため、テスト環境で行うのが安全です。

- **ツールのバージョンに注意:**  
  コマンドの仕様が変更されることがあるため、公式ドキュメントを確認しながら作業してください。

---

# 7. まとめ

tcpdump、tcprewrite、tcpreplayを使うことで、ネットワークの通信データを記録し、変更し、再現することができます。これにより、ネットワークの問題を特定したり、セキュリティ検証を行ったりすることが容易になります。


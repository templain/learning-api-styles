[Table of Contents](../../README.md) | [&larr; API Concepts](../API-CONCEPTS.md) | [Web Protocols &rarr;](../http/README.md)

## ネットワーク

以下は、TCP（Transmission Control Protocol）と TLS（Transport Layer Security）プロトコルを紹介する「ネットワーク」章の補足コンテンツです。

### 必要条件

* [Docker Engine](https://docs.docker.com/engine/install/) または [Docker Desktop](https://docs.docker.com/desktop/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* 必要に応じて [Wireshark](https://www.wireshark.org/)
* Linux または macOS では必要に応じて [screen](https://www.gnu.org/software/screen/)

### ネットワーク実験室のセットアップ

このセクションのセットアップ手順は、本のコードリポジトリのルートから一度だけ実行します。

最初のスクリプトは `client`、`router`、`server` の 3 つの Docker コンテナを作成します。
2 つ目のスクリプトは、`router` を経由して `client` と `server` の間にネットワーク経路を作成します。

```bash
cd src/network
bash scripts/setup_containers.sh
bash scripts/setup_network.sh
```

<details>
<summary>上の例をアニメーション GIF で表示</summary>

[![Example](assets/network-lab-setup-timer.gif)](https://youtu.be/R4IxmkFLJ8U)

</details>

### TCP ECHO サービスの実装

このセクションは、本で説明されている TCP 機能のデモの一つを簡略化したものです。

> [!NOTE]
> TCP ECHO サービスは、クライアントからの接続を待ち受け、接続からデータを読み取り、クライアントが接続を終了するまで、同じデータを接続を介して返すサーバーで構成されます。

#### TCP ECHO サービス

このセクションの目的は、TCP ECHO サービスの機能を実演することです。

```bash
# ターミナル 1 で実行
tmux new-session \; split-window -v \; split-window -v \;

# 上側のペインで実行 <2>
docker compose exec server bash -c "sudo python src/tcp_echo/server.py"

# 真ん中のペインで実行 <3>
CLIENT=netcat
PCAP_FILE=/tmp/tcp_echo_client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -c 12 -w ${PCAP_FILE} 'not icmp and not icmp6' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行 <4>
docker compose exec client bash -c "sudo ip neigh flush all"
docker compose exec client bash -c \
       "(echo -n Hello | netcat -p 8080 -i 1 -q 1 server 8080) && echo"
```

1. tmux で 3 つのペインを作成します。

2. TCP Echo サーバーを起動します。

3. パケットをキャプチャするために `tcpdump` を起動します。
    ここでは `tcpdump` は待機していますが、TCP ECHO サービスのメッセージ交換が完了すると、`tcpdump` は終了し、期待される 12 個のパケットをキャプチャします。
    その結果は _tests_ ディレクトリ内の _tcp_echo_client_netcat.pcap_ ファイルとして保存されます。
    テストディレクトリには、_ *_reference.pcap*_ 形式の参照用 pcap ファイルも含まれています。

4. クライアントを呼び出す前に、3 つ目のペインのウィンドウでクライアント側の ARP（Address Resolution Protocol）キャッシュをクリアします。
    ARP キャッシュには IP アドレスから MAC（Media Access Control）アドレスへの対応が保存されており、ここでクリアすることで、クライアントがキャッシュを使うのではなく ARP 要求を行うことを示せます。
    その後、`netcat` を使って TCP ECHO クライアントを呼び出します。
    クライアントはサーバーから返ってきた "Hello" メッセージを出力して終了することが期待されます。

<details>
<summary>上の例をアニメーション GIF で表示</summary>

[![Example](assets/network-netcat-timer.gif)](https://youtu.be/EldsV3fF3vE)

</details>

クライアントと `tcpdump` の両方が終了した後、`Ctrl+C` を押してサーバーを停止できます。
`tcpdump` によってキャプチャされたフレームを一覧表示します。

```bash
CLIENT=netcat
docker compose exec --no-tty client bash -c "tshark --read-file tests/tcp_echo_client_${CLIENT}.pcap"
```

出力は以下のようなものになります。

```
clientMAC → Broadcast ARP Who has router? Tell client                    # <1>
routerMAC → clientMAC ARP router is at routerMAC                         # <2>
client → server       TCP [SYN]      Seq=0=c0                      Len=0 # <3>
server → client       TCP [SYN, ACK] Seq=0=s0       Ack=1=c0+1     Len=0 # <4>
client → server       TCP [ACK]      Seq=1=c0+1     Ack=1=s0+1     Len=0 # <5>
client → server       TCP [PSH, ACK] Seq=1=c0+1     Ack=1=s0+1     Len=5 # <6>
server → client       TCP [ACK]      Seq=1=s0+1     Ack=6=c0+1+5   Len=0 # <7>
server → client       TCP [PSH, ACK] Seq=1=s0+1     Ack=6=c0+1+5   Len=5 # <8>
client → server       TCP [ACK]      Seq=6=c0+1+5   Ack=6=s0+1+5   Len=0 # <9>
client → server       TCP [FIN, ACK] Seq=6=c0+1+5   Ack=6=s0+1+5   Len=0 # <10>
server → client       TCP [FIN, ACK] Seq=6=s0+1+5   Ack=7=c0+1+5+1 Len=0 # <11>
client → server       TCP [ACK]      Seq=7=c0+1+5+1 Ack=7=s0+1+5+1 Len=0 # <12>
```

1. クライアントはルーティングテーブルを参照してルーターの IP アドレスを取得します。
    その後、ローカルネットワーク上でルーターの MAC アドレス（routerMAC）を特定するために、イーサネット ARP（Address Resolution Protocol）要求をブロードキャストします。
    ARP の目的は、IP アドレスからローカルネットワーク上の MAC アドレスへの対応付けを提供することです。

2. routerMAC は Ethernet を介してクライアントに応答し、ルーターの IP アドレスが routerMAC の MAC アドレスに対応していることを確認します。

3. TCP の 3 ウェイハンドシェイクが開始されます。
    クライアントは、ペイロードサイズが 0 のセグメント（Len=0）を送信し、SYN フラグを設定して TCP 接続を開始し、独自のシーケンス番号（Seq=0）を付与します。
    クライアントの初期シーケンス番号（c0）とサーバーの初期シーケンス番号（s0）はアルゴリズムに従って生成され、通常は 0 ではありません。
    ただし、人間が読みやすいように、`tshark` はそれぞれをクライアントとサーバーの相対的なゼロ参照（Seq=0）として扱います。

4. サーバーはクライアントに対してペイロードサイズが 0 のセグメントを返し、自身のシーケンス番号（Seq=s0）を設定するとともに、クライアントの [phantom byte](https://support.novell.com/techcenter/articles/nc2001_05e.html)（ghost byte とも呼ばれる）を受け取ったことを示すために、クライアントのシーケンス番号に 1 バイト足した値（Ack=c0+1）を返します。
    前述のとおり、`tshark` はこれを Ack=1 として表示します（Ack=c0+1 のような大きな数ではなく）。
    これは、ペイロード長が 0 なのに 1 バイトが確認応答されているように見えるため、やや不自然に思えるかもしれません。
    これは、SYN または FIN フラグが設定されたセグメントは、_phantom byte_ として追加の 1 バイト長として数えられるためです。
    送信されたセグメントには SYN と ACK の両方のフラグが設定されています。

5. クライアントは送信した _phantom byte_ の分だけシーケンス番号を 1 バイト進め（Seq=c0+1）、サーバーの _phantom byte_ を受け取ったことを示すため、サーバーの初期シーケンス番号に 1 バイト足した値（Ack=s0+1）を返します。
    クライアントはこのセグメントを ACK フラグ付きで送信します。
    この時点で TCP の 3 ウェイハンドシェイクは完了し、クライアントはメッセージを送信できる状態になります。

6. クライアントは 5 バイトのメッセージ "Hello"（Len=5）をサーバーに送信します。
    PSH フラグが設定されており、これはサーバー側の TCP スタックがバッファリングせずに直ちにアプリケーションにセグメントを渡すべきであることを示します。

7. サーバーはクライアントから送られた 5 バイトのセグメントを受け取ったことを確認応答します（Ack=c0+1+5）。

8. サーバーはクライアントに対して、同じ 5 バイトの "Hello" メッセージを Len=5 でエコー（返送）します。

9. クライアントは送信した "Hello" メッセージの長さ 5 に応じてシーケンス番号を進め（Seq=c0+1+5）、ACK セグメントを送信します。

10. クライアントは FIN セグメントを送信し、接続を終了したいことを示します。
    これは、クライアントにこれ以上送信するメッセージがない場合に発生します。

11. サーバーはクライアントにエコーした "Hello" メッセージの長さに応じてシーケンス番号を進め、クライアントからこれまでに 7 バイト（Ack=c0+1+5+1）を受け取ったことを確認応答し、FIN フラグを設定して接続の終了を確認します。
    ここには 2 つの _phantom byte_ が含まれています。

12. クライアントはサーバーから受け取った _phantom byte_ に応じて確認応答番号を進め、ACK フラグで接続終了を確認します。

<details>
<summary><strong>Scapy を使った TCP ECHO クライアント</strong></summary>

```bash
# ターミナルで実行
tmux new-session \; split-window -v \; split-window -v \;

# 上側のペインで実行
docker compose exec server bash -c "sudo python src/tcp_echo/server.py"

# 真ん中のペインで実行
CLIENT=scapy
PCAP_FILE=/tmp/tcp_echo_client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -c 12 -w ${PCAP_FILE} 'not icmp and not icmp6' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行
docker compose exec client bash -c "sudo ip neigh flush all"
docker compose exec client bash -c \
        "sudo python src/tcp_echo/client_scapy.py && echo"
```

[![Example](assets/network-scapy-timer.gif)](https://youtu.be/RLrnEGHJ9MY)

</details>

### セキュリティ

暗号化は、今日のネットワーク通信のセキュリティを高める一般的な方法です。
しかし、初期のネットワークプロトコルが開発された当時は、標準化された暗号化が実現できませんでした。
これは、米国とその同盟国が一般市民および他国の暗号技術へのアクセスを制限した、いわゆる [Crypto Wars](https://en.wikipedia.org/wiki/Crypto_Wars)（最近の暗号資産戦争とは異なる）によるものです。
外国からの暗号技術へのアクセス制限は 1990 年代に解除され、その後、SSL/TLS（Secure Sockets Layer/Transport Layer Security）などの Web 暗号化プロトコルが開発されました。

#### 暗号学入門

暗号学は、安全な通信手法を研究する科学分野です。
これには、通信を保護する暗号技術（cryptography）と、セキュリティを破ろうとする暗号解析（cryptanalysis）が含まれます。[^1]

以下の画像のように、TLS は 3 つの暗号学の領域を用います。_非対称暗号_、_対称暗号_、_ハッシュ関数_ です。
これらの領域で確立された暗号プリミティブと呼ばれる手法を用いて、暗号学は機密性、完全性、真正性、否認防止といったセキュリティ目標を達成しようとします。[^2]
これらの目標は、TLS 1.3 の暗号スイート `TLS_AES_256_GCM_SHA384` を文脈に含めて次のセクションで導入・説明します。

<div align="center">
  <img src="assets/cryptology.drawio.png" alt="Cryptography domains used by a TLS version 1.3 cipher suite">
</div>

暗号は通信を安全にするための手法です。
保護対象のメッセージを平文（plaintext）と呼び、安全な形式に変換された後のメッセージを暗号文（ciphertext）と呼びます。
平文を暗号文に変換する処理を暗号化と呼び、暗号文を平文に戻す逆の処理を復号と呼びます。
暗号化と復号の両方には鍵が必要です。
暗号化と復号で異なる鍵を使う場合は非対称、同じ鍵を使う場合は対称です。

最初の通信保護の試みは対称暗号を使っていました。
その例として、Python の Read, Evaluate, Print, Loop（REPL）で示す `Caesar Cipher` があります。
アルファベットを 3 文字ずつずらす暗号です。
数値 3 は鍵とみなせます。
たとえば "abc" は "def" になります。
平文 "Hello" をこの暗号で暗号化すると暗号文 "Khoor" になります。
復号は文字を 3 つ戻すことで行います。

> [!NOTE]
> 暗号化は `機密性` を提供します。
> 機密性は、情報が復号鍵を持つ許可された相手だけにアクセスできることを保証します。

```python
>>> "".join([chr(ord(c) + 3) for c in "Hello"])
"Khoor"

>>> "".join([chr(ord(c) - 3) for c in "Khoor"])
"Hello"
```

> [!TIP]
> Python の REPL は `docker compose exec client python` で起動でき、`exit()` と _ENTER_ を入力して終了できます。

"Khoor" は元の "Hello" と無関係に見えますが、攻撃者はシフト値をすべて試して、もっともらしいメッセージが出るものを見つけることで、シフト値を取り出そうとできます。
この種の攻撃は総当たり攻撃と呼ばれ、暗号文から平文を回復するために可能な鍵をすべて試します。

別の攻撃手法は、与えられた言語における文字の出現頻度分析に基づくものです。
たとえば英語では `e` が最も頻繁に使われ、約 13% の頻度を持ちます。
長い暗号文を持つ攻撃者は、暗号文の中で最も頻繁に出現する文字を `e` に対応させ、その後も同様に他の文字を推測していきます。
この攻撃は、文字を 3 ずらすような暗号だけでなく、任意の文字マッピングを使う暗号にも有効です。

頻度分析やその他の攻撃（例: side-channel attacks）を考慮すると、通信の機密性を完全に保証できるのかという疑問が生じます。
この保証を実現する方法の一つは、暗号アルゴリズムを公表しないことです。
しかし、アルゴリズムを発見・漏えいするインセンティブが最終的にその露見につながる可能性が高いです。
そのため残るのは暗号のもう一つの要素である鍵だけが機密性の提供源となるという考えです。
これは、鍵以外の情報はすべて公開されても安全であるという Kerckhoffs の原理の考え方です。

次に残る問題は、暗号アルゴリズムが公開されている状況で、鍵だけに基づいて機密性をどのように実現できるかということです。
これに最も近いアプローチがワンタイムパスワード（OTP）です。これは次のように構成されます。
アルファベットではなく、平文は 2 進数で表現されます。
XOR（排他的論理和）と呼ばれる数学的演算があり、次の真理値表のように 0 または 1 のビットを等確率で生成します。
平文を本当にランダム[^3] な鍵で XOR し、その鍵を再利用しなければ、攻撃者が平文の各ビットを当てる確率は 50% を超えることはありません。

<div align="center">

| A   | B   | A XOR B |
| --- | --- | ------- |
| 0   | 0   |    0    |
| 0   | 1   |    1    |
| 1   | 0   |    1    |
| 1   | 1   |    0    |

</div>

XOR にはもう一つ便利な性質があります。XOR を 2 回適用すると恒等演算になることです。
つまり、暗号化では平文と鍵のビットごとの XOR を行い、復号ではもう一度 XOR を適用するだけでよいのです。
以下はこの使い方の例です。

```python
>>> import random
>>> random.seed(42)  # <1>
>>> plaintext = format(ord("H"), "07b")  # <2>
>>> plaintext
"1001000"

>>> key = format(random.randint(0, 127), "07b")  # <3>
>>> key
"0011100"

>>> ciphertext = "".join([str(int(c) ^ int(k)) for c, k in zip(plaintext, key)]) # <4>
>>> ciphertext
"1010100"

>>> "".join([str(int(c) ^ int(k)) for c, k in zip(ciphertext, key)])  # <5>
"1001000"
```

1. この行は擬似乱数生成器のシードを設定し、複数回実行しても同じ挙動になるようにしています。
    これにより、ここで示した結果と同じものが得られます。
    なお、真にランダムな数ではなく擬似乱数を使うのは OTP の原理に反します。
    定義上、アルゴリズムの結果である数は真にランダムではありません。
    しかし、ほとんどのアプリケーションでは擬似乱数を使うことは許容されます。

2. `format` 関数は大文字の 7 ビット ASCII 文字 "H" を 7 ビットの文字列に変換します。

3. `randint` 関数は対称鍵として使う 7 ビットのランダムな文字列を生成します。

4. このリスト内包表は平文と鍵のビットごとの XOR（^）を実行し、平文を暗号文へ暗号化します。

5. このリスト内包表の出力は、暗号文に鍵を XOR して再度ビットごとに復号すると元の平文に戻ることを確認しています。

> [!WARNING]
> Python の `random` モジュールは暗号用途には使うべきではありません。
> 代わりに `secrets` のような、暗号学的に安全な擬似乱数生成器専用のモジュールを使うべきです。

OTP は平文をビットごとに暗号化するストリーム暗号の例です。
OTP に対する厳密な要件から、これは安全ではある一方で扱いにくい構造であることが分かります。鍵は平文と同じ長さであり、真にランダムで、暗号文の受信者全員と安全に共有される必要があります。
1 GB の動画をオンラインで視聴する前に、放射線崩壊のような物理的にランダムとみなされるプロセスで 1 GB の鍵を生成し、その鍵を安全な経路でオンラインサービスに渡す必要があると想像してみてください。たとえば現地まで旅行して代表者に直接鍵を手渡す必要があるでしょう。

このため、OTP の代わりに専用の対称暗号が使われ、最も一般的なのは [Advanced Encryption Standard](https://doi.org/10.6028%2FNIST.FIPS.197)（AES）[^4] です。
その構造により、同じ鍵を使って複数の平文ブロックを暗号化できます。
これは、OTP と同様に同じ 2 つの平文ブロックが異なる暗号文になるように、nonce（一度だけ使う数）という追加のランダム要素を導入することで実現されます。
さらに AES の特殊な動作モードである Gallois counter mode（GCM）[^5] を使うと、鍵を持つ暗号文の受信者が、受信したメッセージの完全性と真正性を検証できます。

> [!NOTE]
> `完全性` は情報の正確さと完全性を検証できることを保証し、`真正性` は受信側がその情報が指定された送信者から来たことを確認できることを意味します。

この時点で、TLS 1.3 の暗号スイート `AES_256_GCM` を含む `TLS_AES_256_GCM_SHA384` の役割の意図を理解できるようになります。
これは、TLS で交換されるアプリケーションデータに対して、対称暗号化によって機密性、完全性、真正性を提供します。
この暗号コンポーネントの役割は、実際のパケットキャプチャを使って [TCP ECHO Client with OpenSSL](#tcp-echo-client-with-openssl) セクションでさらに説明します。

残念ながら、対称暗号には欠点があります。
対称暗号の大きな問題は、通信に関わるすべての当事者が同じ鍵を共有する必要があることです。
そのため鍵配送のための安全な手段が必要になり、そこでも機密性が必要になるため循環論に陥ります。
したがって、対称暗号には鍵配送用の別の安全なチャネルが必要です。
このチャネルの数は通信相手の数に応じて二次的に増加します。
たとえば、会社の `n` 人の従業員がいる場合、`n * (n - 1) / 2` 個のチャネルを確立する必要があり、それぞれの従業員が `n - 1` 個の鍵を管理することになります。
1000 人の会社では、ほぼ 50 万回の鍵交換が必要になります。
さらに、対称暗号は否認防止を提供できません。

> [!NOTE]
> `否認防止` は、通信に参加する任意の当事者が情報をその発信元に帰属させられることを保証します。
> 定義から、暗号方式が否認防止を提供するなら、それは認証も提供することになります。
>
> 対称暗号は共有鍵に依存するため、暗号文を単一の人物に一意に帰属させることができず、否認防止を提供できません。

良いニュースは、非対称暗号がこれらの問題を解決することです。
それは安全な鍵配送チャネルを確立し、否認防止も提供します。
では、非対称暗号がどのようにしてこれを実現するのかを見ていきましょう。

非対称暗号学では、公開鍵暗号学とも呼ばれ、各当事者は公開鍵と秘密鍵のペアを持ちます。
公開鍵は公開してもよく、秘密鍵は誰とも共有しません。
銀行口座の例が、この考えを理解するのに役立ちます。
口座番号は公開鍵に相当します。
この番号を知っている人なら誰でもその口座に送金できます。
一方で、口座の所有者であるあなただけが、PIN やパスワードのような秘密情報を使ってお金を引き出せます。これは秘密鍵に相当します。

一方で、対称暗号は金庫のようなものです。
金庫の組み合わせ（鍵）を知っている人だけが開けられます。
これは、対称暗号で暗号化と復号に同じ鍵を使うことに似ています。

DH は、安全でない通信チャネル上で共有秘密を確立するために使われる一般的な公開鍵暗号プロトコルです。
DH は 1976 年に発表した Whitfield Diffie と Martin Hellman の姓を取って名付けられています。[^6]
DH はこの鍵確立方式を指す頭字語として使われます。
DH 鍵共有の数学を理解するのに役立つ数値例を、図と Python 対話セッションの両方で示します。[^7]
このプロトコルのデモでは、公開鍵暗号学の説明でよく使われる 2 つの通信当事者、Alice と Bob を使います。[^8]

<div align="center">
  <img src="assets/DH.drawio.png" alt="Diffie-Hellman key establishment">
</div>

```python
>>> g, p = 5, 23      # <1>

>>> a = 4             # <2>
>>> A = (g ** a) % p  # <3>
>>> A
4

>>> b = 3             # <4>
>>> B = (g ** b) % p  # <5>
>>> B
10

>>> s = (B ** a) % p  # <6>
>>> s
18

>>> s = (A ** b) % p  # <7>
>>> s
18
```

1. このプロトコルでは大きな素数 `p` と、それに対応する数 `g` が必要です。`g` は `p` を法とする整数の乗法群の生成元です。
    これらの数は公開されています。
    このデモでは、可読性のために小さな素数 `p` を使っています。[^9]

2. 通信の一方、Alice と呼ばれる側がランダムな整数 `a` を生成します。これは Alice の秘密鍵です。

3. Alice は秘密鍵から公開鍵 `A` を計算します。これは生成元 `g` を秘密鍵 `a` 乗し、`p` で割った余りです。
    このデモでは秘密鍵と公開鍵がどちらも 4 になるのは偶然です。
    Alice は公開鍵を公開通信路を通じて通信相手である Bob に送ります。

4. Bob も同様の手順を行って秘密鍵と公開鍵を作成します。まず、ランダムに秘密鍵 `b` を生成します。

5. その後、Bob は公開鍵 `B` を計算し、それを Alice に送ります。

6. Alice は Bob の公開鍵 `B` を自分の秘密鍵 `a` でべき乗し、`p` を法として共有秘密 `s` を計算します。

7. Bob も同様に Alice の公開鍵と自分の秘密鍵を使って共有秘密 `s` を計算します。
    DH 鍵共有の数学では、Alice と Bob が通信路を介して共有秘密そのものを交換せずに、同じ共有秘密の値を持つことが保証されます。
    数値として `(B ** a) % p == (((g ** b) % p) ** a) % p == ((g ** b) ** a) % p == (g ** (b*a)) % p == (g ** (a*b)) % p == ((g ** a) ** b) % p == (((g ** a) % p) ** b) % p == (A ** b) % p` であることを確認できます。

攻撃者が公開されている公開鍵 `A`、素数 `p`、生成元 `g` から Alice の秘密鍵 `a` を計算できないのはなぜかと疑問に思うかもしれません。
答えは対数の数学的性質にあります。対数は指数計算の逆演算です。
DH の安全性は、大きな整数に対する離散対数を計算することの計算困難さに依存しており、これは離散指数計算よりもはるかに負荷が高いです。[^10]

DH は対称暗号が必要とする鍵配送チャネルの問題に対する解決策を提供します。
実際には対称鍵そのものは交換されず、公開鍵だけが通信路を通じて交換されます。

[目次](../../README.md) | [← ネットワーク](../network/README.md) | [REST →](../django/docs/REST.md)

## Web Protocols

以下は、Web Protocols の章に付随する補足コンテンツです。

### 必要条件

* [Docker Engine](https://docs.docker.com/engine/install/) または [Docker Desktop](https://docs.docker.com/desktop/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* 必要に応じて [Wireshark](https://www.wireshark.org/)
* Linux または macOS では、必要に応じて [screen](https://www.gnu.org/software/screen/)

### HTTP Lab のセットアップ

このセクションのセットアップ手順は、書籍のコードリポジトリのルートから一度だけ実行します。
このスクリプトは、`client`、`http-server`、`https-server` の 3 つの Docker コンテナを作成します。

```bash
cd src/http
bash scripts/setup_containers.sh
```
<details>
<summary>上の例をアニメーション GIF で表示</summary>

[![Example](assets/http-lab-setup-timer.gif)](https://youtu.be/h5FZtkVNGlc)

</details>

</br>

<details>
<summary><strong>HTTP/0.9</strong></summary>

この例では、`tcpdump` で取得した `client` と `server` 間の HTTP/0.9 通信を示します。
取得した通信は、_tests_ ディレクトリにある _client_http0.9.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行
tmux new-session \; split-window -v \;

# 上側のペインで実行
CLIENT=http0.9
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -w ${PCAP_FILE} 'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行
docker compose exec client bash -c \
       "echo -en 'GET /Hello.html\r\n\r\n' | netcat -p 8080 http-server 80"
```

[![Example](assets/http-0.9-timer.gif)](https://youtu.be/sz5J0oEyFic)

</details>

<details>
<summary><strong>HTTP/1.0</strong></summary>

この例では、`tcpdump` で取得した `client` と `server` 間の HTTP/1.0 通信を示します。
取得した通信は、_tests_ ディレクトリにある _client_http1.0.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行
tmux new-session \; split-window -v \;

# 上側のペインで実行
CLIENT=http1.0
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -w ${PCAP_FILE} 'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行
docker compose exec client bash -c \
       "echo -en 'GET /HelloValid.html HTTP/1.0\r\n\r\n' | \
       netcat -p 8080 http-server 80"
```

[![Example](assets/http-1.0-timer.gif)](https://youtu.be/ZpUkT4m2UVk)

</details>

### HTTP/1.1

このセクションの目的は、`HTTP/1.1` における TCP 接続の持続性を示すことです。
実際に確認しながら、HTTP pipelining を見ていきます。これは接続の持続性に依存するからです。
その過程で、HTTP/1.1 のヘッダーに現れる他の興味深い機能も確認できます。

この例では `tcpdump` を使ってネットワークトラフィックをキャプチャします。
取得した通信は、_tests_ ディレクトリにある _client_http1.1.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行 <1>
tmux new-session \; split-window -v \;

# 上側のペインで実行 <2>
CLIENT=http1.1
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -w ${PCAP_FILE} \
       'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行 <3>
docker compose exec client bash -c \
       'GET="GET / HTTP/1.1\r\nHost:host\r\n" && echo -en \
       "${GET}Connection:keep-alive\r\n\r\n${GET}Connection:close\r\n\r\n" | \
       netcat -p 8080 http-server 80'
```

1. tmux で 2 つのペインを作成します。

2. `tcpdump` を使って新しいパケットキャプチャを開始します。取得したトラフィックは、_tests_ ディレクトリにある _client_http1.1.pcap_ ファイルとして保存されます。

3. `netcat` を使って、サーバーに 2 つの連続した pipelined な `HTTP "GET /"` リクエストを送信します。最初のリクエストでは、`HTTP/1.1` ではデフォルトで暗黙的に有効な `"Connection: keep-alive"` を明示的に指定しています（ヘッダーを削除して確認してください）。2 つ目のリクエストでは `"Connection: close"` を使い、サーバーは文書全体を送信した後に接続を閉じます。これは HTTP/1.1 以前のデフォルト動作です。ここで使う `"Host: host"` ヘッダーの `host` の値は任意です。サーバーにはデフォルトの virtualhost が設定されているためです。ただし、このヘッダーは HTTP/1.1 のリクエストでは必須です。ヘッダーを省略した場合に仕様どおりに動作するか、試して確認してみてください！

<details>
<summary>上の例をアニメーション GIF で表示</summary>

[![Example](assets/http-1.1-timer.gif)](https://youtu.be/foducq9Nq1s)

</details>

クライアントは、デフォルトの `http-server` レスポンスを 2 回出力した後、"It works!" を含む HTML 文書を表示して終了することが期待されます。期待される出力は以下のとおりです。

```
HTTP/1.1 200 OK                              # <1>
Date: Tue, 23 Jul 2024 16:47:14 GMT
Server: Apache/2.4.61 (Unix)
Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
ETag: "2d-432a5e4a73a80"                     # <2>
Accept-Ranges: bytes                         # <3>
Content-Length: 45
Keep-Alive: timeout=5, max=100               # <4>
Connection: Keep-Alive                       # <5>
Content-Type: text/html

<html><body><h1>It works!</h1></body></html>
HTTP/1.1 200 OK
Date: Tue, 23 Jul 2024 16:47:15 GMT
Server: Apache/2.4.61 (Unix)
Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
ETag: "2d-432a5e4a73a80"                     # <6>
Accept-Ranges: bytes
Content-Length: 45
Connection: close                            # <7>
Content-Type: text/html

<html><body><h1>It works!</h1></body></html>
```

1. サーバーからのレスポンスに含まれる HTTP 200 (OK) ステータスコードは、クライアントのリクエストが成功したことを示します。すべてのレスポンス行は、不可視の <CR><LF> シーケンスで終わることを忘れないでください。

2. `ETag` ヘッダーは _Entity Tag_ に由来します。_Entity_ は文書リソースを指し、この例では 2007 年のデフォルトの _index.html_ ファイル（Apache が古いコンテンツを配信している！）です。ETag は HTML ファイルのサイズや更新日時などのリソース情報に依存し、キャッシュを支援するために使われます。

3. `Accept-Ranges` ヘッダーは、サーバーが部分コンテンツのリクエストをサポートしていることを示します。これにより、クライアントはファイルの一部を要求したり、ダウンロードを再試行したりできます。

4. `Keep-Alive` ヘッダーは、サーバーの永続接続の制限をクライアントに通知します。この場合、サーバーは同じ接続上で次のリクエストを 5 秒待ち、1 つの接続で最大 100 件のリクエストを受け付けます。

5. `Connection` ヘッダーの `Keep-Alive` の値は、サーバーが文書全体をクライアントに送信した後に接続を維持するよう指示します。

6. `ETag` ヘッダーの値が変わっていないことは、前回のクライアントリクエスト以降に _index.html_ ファイルが変更されていないことを示しています。

7. `Connection` ヘッダーの `close` の値は、サーバーが文書全体を送信した後に接続を閉じるよう指示します。つまり、HTTP/1.1 より前の HTTP と同じ挙動です。

`tcpdump` は _Ctrl+C_ で停止し、保存されたパケットキャプチャを `tshark` で読み、TCP 接続の持続性を確認します。

```bash
CLIENT=http1.1
docker compose exec --no-tty client bash -c "tshark --read-file tests/client_${CLIENT}.pcap"
```

次のようなシーケンスが表示されます。

```
client → server TCP  [SYN]      Seq=0           Len=0 # <1>
server → client TCP  [SYN, ACK] Seq=0   Ack=1   Len=0 # <1>
client → server TCP  [ACK]      Seq=1   Ack=1   Len=0 # <1>
client → server HTTP GET / HTTP/1.1 GET / HTTP/1.1    # <2>
server → client TCP  [ACK]      Seq=1   Ack=100 Len=0 # <3>
server → client HTTP HTTP/1.1 200 OK  (text/html)     # <4>
client → server TCP  [ACK]      Seq=100 Ack=327 Len=0
server → client HTTP HTTP/1.1 200 OK  (text/html)     # <5>
client → server TCP  [ACK]      Seq=100 Ack=616 Len=0
server → client TCP  [FIN, ACK] Seq=616 Ack=100 Len=0 # <6>
client → server TCP  [FIN, ACK] Seq=100 Ack=617 Len=0 # <6>
server → client TCP  [ACK]      Seq=617 Ack=101 Len=0 # <6>
```

1. クライアントからサーバーへの TCP 3-way handshake が確立されます。

2. クライアントは、デフォルトの `/` 文書を取得するために 2 つの pipelined HTTP リクエストを送信します。

3. サーバーは、クライアントからの HTTP リクエストを受信したことを確認します。

4. サーバーは最初の HTTP レスポンスをクライアントに送信します。サーバーは TCP 接続を開いたままにします。

5. サーバーは 2 つ目の HTTP レスポンスをクライアントに送信します。

6. サーバーは HTML 文書全体をクライアントに送信した後、TCP 接続の終了を開始し、接続は終了します。

上のパケットシーケンスから、HTTP/1.1 では TCP 接続の持続性によって、その後の HTTP リクエストにおける 3-way handshake の RTT を回避でき、レイテンシを低減できることが確認できます。

### DNS request/response

このセクションの目的は、DNS トラフィックをキャプチャする方法を示すことです。

この例では `tcpdump` を使ってネットワークトラフィックをキャプチャします。
取得した通信は、_tests_ ディレクトリにある _client_dns.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行 <1>
tmux new-session \; split-window -v \;

# 上側のペインで実行 <2>
CLIENT=dns
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -c 2 -w ${PCAP_FILE} 'port 53' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行 <3>
docker compose exec client bash -c "dig +short example.com @1.1.1.1"
```

1. tmux で 2 つのペインを作成します。

2. `tcpdump` を使ってクライアント上で新しいパケットキャプチャを開始します。
`tcpdump` は、期待される 2 パケットをキャプチャした後に終了し、取得したトラフィックは _tests_ ディレクトリの _client_dns.pcap_ ファイルとして保存されます。

3. `1.1.1.1` の DNS サーバーに対して、`example.com` サーバーの IP アドレスを問い合わせます。

保存したパケットキャプチャを `tshark` で読みます。

```bash
CLIENT=dns
docker compose exec --no-tty client bash -c "tshark --read-file tests/client_${CLIENT}.pcap"
```

次のようなシーケンスが表示されます。

```
192.168.114.2 → 1.1.1.1       DNS query example.com
      1.1.1.1 → 192.168.114.2 DNS query response example.com 93.184.215.14
```

`192.168.114.2` はクライアントの IP アドレス、`1.1.1.1` は DNS サーバーの IP アドレス、`93.184.215.14`（環境によって異なります）は _example.com_ HTTP サーバーの IP アドレスです。

<details>
<summary><strong>HTTP in a Browser</strong></summary>

この例では、Web ブラウザーが HTTP リクエストをどのように発行するかを示します。
この例では `tcpdump` を使って、Web ブラウザーとサーバーの間の通信をキャプチャします。
取得した通信は、_tests_ ディレクトリにある _client_firefox.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行
tmux new-session \; split-window -v \;

# 上側のペインで実行
CLIENT=firefox
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump -w ${PCAP_FILE} 'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行
docker compose exec client bash -c \
       "rm -rf ~/.cache/mozilla/firefox/* && \
       rm -rf ~/.mozilla/firefox/*.profile && \
       firefox --headless --screenshot /tmp/website-in-firefox.png \
       http://http-server/HelloWeb.html && \
       cp /tmp/website-in-firefox.png tests"
```

[![Example](assets/http-in-a-browser-timer.gif)](https://youtu.be/jxJVP374ePw)

</details>

### TCP の制限

元の HTTP/0.9 プロトコルは TCP の上で動作するように設計されました。
そのため HTTP は TCP の信頼性や順序保証などの機能の恩恵を受けますが、同時に TCP の制限にも影響を受けます。

HTTP の性能に悪影響を与える TCP の特性として、_TCP Head-of-line blocking_ と _TCP slow start and congestion avoidance_ の 2 つが特に重要です。
HTTP/2 や HTTP/3 のような最新の HTTP では、これらの問題を軽減する最適化が導入されています。

<details>
<summary><strong>TCP Head-of-line Blocking</strong></summary>

この例では、TCP の head-of-line blocking を示します。
この例では、`tcpdump` を使ってクライアントとサーバーの間のネットワークトラフィックをキャプチャします。
取得した通信は、_tests_ ディレクトリにある _client_tcp.pcap_ ファイルとして保存されます。

```bash
# ターミナルで実行
tmux new-session \; split-window -v \; split-window -v \;

# 上側のペインで実行
docker compose exec client bash -c "netcat -l -p 80 && echo"

# 中央のペインで実行
CLIENT=tcp
PCAP_FILE=/tmp/client_${CLIENT}.pcap
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump --interface lo -w ${PCAP_FILE} 'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行
docker compose exec client bash -c \
       "sudo tc qdisc del dev lo root || true && \
       sudo tc qdisc add dev lo root handle 1: prio && \
       sudo tc qdisc add dev lo parent 1:1 netem delay 5s && \
       sudo tc filter add dev lo protocol ip parent 1:0 prio 1 handle 1 fw flowid 1:1 && \
       tc qdisc show dev lo && \
       sudo ip link set dev lo mtu 1500 && \
       ip a && \
       sudo iptables -A OUTPUT -p tcp --dport 80 \
         -m string --string DELAYME --algo bm -j MARK --set-mark 1 && \
       sudo iptables-legacy-save && sudo iptables-save"
docker compose exec client bash -c \
       "exec 3<>/dev/tcp/127.0.0.1/80 && \
       echo -n SEGMENT1 >&3 && \
       echo -n DELAYME >&3 && \
       echo -n SEGMENT11 >&3 && \
       sleep 1 && \
       echo -n SEGMENT111 >&3 && \
       exec 3<&- && \
       exec 3>&- && \
       sleep 5"
```

[![Example](assets/http-TCP-HOL-timer.gif)](https://youtu.be/JaUxwZG5Nd4)

</details>

#### TCP Slow Start と Congestion Avoidance

TCP は、与えられたネットワーク条件下で可能な最大帯域を実現し、またネットワークを共有する機器間で公平性を保つために、輻輳制御を実装しています。
主要な輻輳制御メカニズム[^1] には、_TCP slow start_ と _congestion avoidance_ が含まれます。
TCP slow start では、送信側が受信側から ACK を受け取るたびに、1 RTT の中で送信するセグメント数を増やしてネットワークを探る方式です。
この増加は通常乗法的であり、例えば 2 つのセグメントから始め、ACK を受け取った後に 4 つのセグメントを送信し、その後さらに増やす、という流れになります。
パケット損失が検出された場合、送信側は送信量を減らします（バックオフ）。
なお、TCP 接続の両端は独立に輻輳制御を行います。

この手法の結果として、TCP は単位時間あたりの送信量を急速に増やします。
その後、_slow start threshold_（_ssthresh_）に達すると、輻輳制御アルゴリズムはあまり貪欲ではない _congestion avoidance_ フェーズに切り替わります。
TCP slow start フェーズは名前の通り「遅い」ように見えますが、輻輳回避フェーズよりも帯域増加が速いです。
また、輻輳制御には時間がかかることが明確であり、その結果、小さなサイズのリソースをブラウザーが取得するような短命な TCP 接続では性能が悪くなります。

以下の例では、帯域幅測定ツール [iperf](https://sourceforge.net/projects/iperf2/) を使って TCP の輻輳制御を確認します。3 つのペインを使う必要があります。
なお、結果は環境によって異なります。
以下のコードには、この実験の注釈付きの実行フローが含まれています。

```bash
# ターミナルで実行 <1>
tmux new-session \; split-window -v \; split-window -v \;

# 上側のペインで実行 <2>
docker compose exec client bash -c "iperf --server --port 80 --time 10"

# 中央のペインで実行 <3>
CLIENT=iperf
docker compose exec client bash -c \
       "sudo rm --force ${PCAP_FILE} && \
       sudo tcpdump --interface lo -w ${PCAP_FILE} 'port 80' && \
       cp ${PCAP_FILE} tests"

# 下側のペインで実行 <4>
docker compose exec client bash -c \
       "sudo tc qdisc del dev lo root || true && \
       sudo tc qdisc add dev lo root netem delay 100ms rate 100mbit loss 1% && \
       tc qdisc show dev lo && \
       sudo ip link set dev lo mtu 1500 && \
       ip a"
docker compose exec client bash -c \
       "iperf --client 127.0.0.1 --bind 127.0.0.1:8080 --port 80 \
       --interval 1 --time 5"
```

1. tmux で 3 つのペインを作成します。

2. `iperf` サーバーを起動します。
サーバーは _client_ コンテナ内で起動し、_loopback_ ネットワークインターフェースを通じてクライアントと通信します。
サーバーは設定したタイムアウト後にブロックしたまま終了します。

3. `tcpdump` を使って新しいパケットキャプチャを開始します。
取得したトラフィックは、_tests_ ディレクトリにある _client_iperf.pcap_ ファイルとして保存されます。

4. `iperf` クライアントを起動します。
クライアントは設定した時間だけトラフィックを送信して終了します。
なお、_loopback_ インターフェースのネットワーク性能特性は、_tc_ ツールを使って変更されます。

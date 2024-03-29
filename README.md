# wges
 WireGuard Easy Setup

## １　概要

　WireGuardの設定ファイルを手軽に作成するBashスクリプトです。特に次のような用途に適しています。
- WireGuardサーバー1台に1台または複数のクライアントが接続
- 外部からNAT内側のLANに接続、またはサーバーを踏み台にしたインターネット接続

　通信に必要な公開鍵・秘密鍵を、接続数分だけ自動的に生成します。IPv6にも対応し、仮想インターフェイスのアドレスもRFC4193に従い設定します。IPv4 over IPv6やIPv6 over IPv4も、環境が整っていれば実現可能です。<br>
　実行すると、スクリプトを置いた場所の直下のwges-outputディレクトリ（場所は変更可能）に次の設定ファイルを生成します。

1. サーバー側設定ファイル
> wg0.conf
2. クライアント側設定ファイル
> c0001.conf, c0002.conf, c0003.conf, …
3. クライアント側設定ファイルをQRコード化したテキスト・画像ファイル（qrencodeがインストールされている場合）
> qr0001.txt, qr0001.png, qr0002.txt, qr0002.png, …

## ２　注意点
　WireGuard自体はPeer to Peerでつなぐだけでトンネリング両端のホストの扱いに差はありませんが、本スクリプトでは、1つのサーバーに多数のクライアントが接続するようなVPNシステムとしての使い方を前提としています。また、設定ファイルを生成するだけなので、それらを適切な場所に設置したり、外部からアクセスするためのDDNSやポート開放等の設定、WireGuardのインストール・起動等は別に行う必要があります。スクリプトの動作はUbuntu22.04のみで確認していますが、最近のメジャーなUnixでBashさえあれば、そのまままたは微修正で動くのではないかと思います。

## ３　使用法
　WireGuardサーバーを置くマシン上で作業することを前提に、使用法を解説します。
### 3．1 スクリプト内変数の修正
　まず、***本スクリプトを直接修正***して下記変数を設定します。
 
>#### Peers
>　サーバーにぶら下がるクライアント数。9999まで対応。
>#### ServerPort
>　サーバーのWireGuardが使用する実ポート。
>#### Endpoint
>　外部からアクセスする時に使用するサーバー名・ポート番号。このアドレスが指す先がIPv6なら「over IPv6」で通信。
>#### EthernetInterface
>　サーバーから他のコンピューターへのアクセスに使用するインターフェイス。
>#### DNS
>　トンネル接続後にクライアント側が使用するネームサーバー。
>#### MTU
>　トンネル内のMTU値を設定する場合（つながらない場合は1350程度に下げてみる）
>#### ClientAllowedIPs
>　クライアント側において、どこ向けのアクセスをトンネルに流すかを指定する変数。$iをクライアント番号、$IPv6PrefixをIPv6プレフィックスとして記述可能<br>
>　この設定は重要。トンネル確立後に、トンネルに流す（かつ受け入れる）相手先アドレスを「,」で区切って記述。例えば次のようなネットワーク。
>1. 192.168.XX.XX/24（サーバー側のローカルLAN）
>2. 10.0.0.0/16,$IPv6Prefix::96（WireGuardで作成された全サーバー・クライアントの仮想インターフェイス）
>3. 0.0.0.0/0, ::/0（全てのネットワーク）
>
>外部からLANへのアクセスを確保するのが目的ならば1、WireGuardでつながったホスト同士でも通信するなら1と2を併記、VPN踏み台サービスのようなことをさせるなら3のように指定。
>
>**（多くの場合、ここより下の変数は変更不要）**
>#### ServerConfigFile
>　サーバーの設定ファイル名。/etc/wireguardにコピーして使用。拡張子を除いたものがサーバー側の仮想インターフェイス名になる。
>#### ServerWgAddress, ClientWgAddress
>　それぞれ、サーバー側・クライアント側の仮想インターフェイスのIPアドレス。$iをクライアント番号、$IPv6PrefixをIPv6プレフィックスとして記述可能。
>#### ServerAllowedIPs
>　サーバー側において、どこ向けのアクセスをトンネルに流すかを指定する変数。$iをクライアント番号、$IPv6PrefixをIPv6プレフィックスとして記述可能。デフォルトの設定では相手クラアント向けアクセスのみをトンネル。
>#### UsePSK
>　trueの場合は事前共有鍵を生成。
>#### OutputDir
>　ファイルの出力ディレクトリを、絶対パスまたは本スクリプトからの相対パスで指定。

### 3．2 スクリプトの実行
スクリプト内でwgコマンドを使うため、実行前に***WireGuardのインストール***を済ませておいてください。

> sudo apt install wireguard

　適当なディレクトリに本スクリプトを置き実行します。
> ./wges.sh

すると、wges-outputディレクトリ下にwg0.confが生成されるので、これを/etc/wireguard/にコピーし
> sudo wg-quick up wg0

を実行することにより、WireGuardが起動します。クライアントからサーバを経由してそれ以外のホストにアクセスするためには/etc/sysctl.confの該当部分を
>net.ipv4.ip_forward=1<br>
>net.ipv6.conf.all.forward=1

と修正した上で
> sudo sysctl -p

を実行します。<br>
　一方クライアント側には、c0001.conf等のファイルをインストールしてください。linuxならばwg0.conf等に名前を変えて(変えなくても良い)/etc/wireguardにコピー、スマホの場合はqr0001.txtやqr0001.pngをWireGuardアプリ内のカメラで読み込ませてください。<br>
　その他の設定に問題がなければ、トンネリングを利用した通信が可能になります。通信を確認したらサーバー側で
> sudo systemctl start wg-quick@wg0

を実行し自動起動を設定します。

## ４　補足
　スクリプトを実行すると、まず$OutputDir/keys下にサーバー・各クライアントの秘密鍵・公開鍵・事前共有鍵を記述したファイルを生成し、次にそれらのファイルを読み込んで$OutputDirに設定ファイルを作るという2段階の手順を踏みます。スクリプトの実行が2度目以降の場合、1段階目で作ったファイルが残っている場合は再利用し、存在しないもののみ生成します。そのため、鍵を作り直す場合はkeys以下のファイルを削除してから実行してください。鍵はそのままでその他の設定を変更したい場合は、削除せずに実行してください。 例えばサーバーの設定のみを修正するときに、keys以下のファイルをそのままにしておくことにより、クライアント全ての設定ファイルをインストールし直すようなことが避けられます。生成したIPv6のプレフィックスも1段階目でサーバーの鍵ファイル(keys/server.txt)に記述するため、これを変更する場合はkeys/server.txtを削除してから実行してください。<br>
　仮想インターフェイスのアドレスは、クライアント番号を4桁として、サーバーが192.168.100.1/16(IPv6 $IPv6Prefix::a000/96)、クライアントが192.168.「クライアント番号の上2桁」.「クライアント番号の下2桁」(IPv6 $IPv6Prefix:1::クライアント番号)に設定されます。クライアント数が9999以下ならば問題は起きないはずですが、別のアドレスを使用する場合はServer(Client)WgAddressを修正してください。その際は、Server(client)AlllowedIPsの修正も必要になるかと思われます。

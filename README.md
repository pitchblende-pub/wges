# wges
 WireGuard Easy Setup

## 概要

　WireGuardの設定がいろいろと面倒なので、設定ファイルを自動生成するBashスクリプトを作りました。似たものは複数ありましたが、自分の用途に合うものが無く自作しました。通信に必要な公開鍵・秘密鍵なども自動的に生成します。IPv6にも対応し、仮想インターフェイスのアドレスもRFC4193に従い設定します。IPv4 over IPv6やIPv6 over IPv4も環境が整っていれば実現可能です。なお、実行はUbuntu22.04で確認していますが、最近のメジャーなUnixなら大体動くのではないかと思います。
 
　実行すると、スクリプトを置いた場所の直下のwges-outputディレクトリ（場所は変更可能）に次の設定ファイルを生成します。
1. サーバー側設定ファイル
> wg0.conf
2. クライアント側設定ファイル
> c0001.conf, c0002.conf, c0003.conf, …
3. クライアント側設定ファイルをQRコード化したテキスト・画像ファイル（qrencodeがインストールされている場合）
> qr0001.txt, qr0001.png, qr0002.txt, qr0002.png, …

## 注意点
　WireGuardではトンネリング両端のホストの扱いに差はありませんが、本スクリプトは、1つのサーバーに多数のクライアントが接続するようなVPNシステムの考え方を前提としています。また設定ファイルを生成するだけなので、それらを適切な場所に設置したり、外部からアクセスするためのDDNSやポート開放等の設定、WireGuardのインストール・起動・自動起動等は別に行う必要があります。

## 使用法
　適当なディレクトリに本スクリプトを置き、***後記の「設定が必要な変数一覧」を編集し***実行してください。wges-outputディレクトリ下にwg0.confが生成されるので、これを/etc/wireguard/にコピーし
> wq-quick up wg0

を実行することにより、WireGuardが起動します。サーバー以外のホストにアクセスするためには/etc/sysctl.confを
>net.ipv4.ip_forward=1<br>
>net.ipv6.conf.all.forward=1

と修正した上でsudo sysctl -pを実行します。

　一方クライアント側には、c0001.conf等のファイルをインストールしてください。linuxならばwg0.conf等に名前を変えて/etc/wireguardにコピー、スマホではqr0001.txtやqr0001.pngを、WireGuardアプリ内のカメラで読み込ませてください。

　その他の設定に問題がなければ、トンネリングを利用した通信が可能になります。通信を確認したらサーバー側で
> sudo systemctl start wg-quick@wg0

を実行し自動起動を設定します。

## 設定が必要な変数一覧
#### Peers
サーバーにぶら下がるクライアント数。9999まで対応。
#### ServerConfigFile
サーバーの設定ファイル。/etc/wireguardにコピーして使用。
#### ServerPort
サーバーのWireGuardが使用する実ポート。
#### Endpoint
外部からアクセスする時に使用するサーバー名・ポート番号。このアドレスが指す先がIPv6なら「over IPv6」で通信。
#### EthernetInterface
サーバーから他のコンピューターへのアクセスに使用するインターフェイス。
#### DNS
トンネル接続後にクライアント側が使用するネームサーバー。
#### ServerWgAddress, ClientWgAddress
それぞれ、サーバー側・クライアント側の仮想インターフェイスのIPアドレス。$iをクライアント番号、$IPv6PrefixをIPv6プレフィックスとして記述可能。
#### ServerAllowedIPs, ClientAllowedIPs
それぞれ、サーバー側・クライアント側において、どの宛先向けのアクセスをトンネリングに流すかを指定する変数。$iをクライアント番号、$IPv6PrefixをIPv6プレフィックスとして記述可能。デフォルトの設定では、サーバー側は相手のクライアント向けアクセスのみをトンネル。クライアント側は、サーバー、サーバーにつながる全クライアント、サーバーがつながっているLAN(192.168.1.0/24)向けをトンネル。***（最後のLANアドレスは環境に従って要修正。）***
#### UsePSK
trueの場合は事前共有鍵を生成。
#### OutputDir
ファイルの出力ディレクトリを、絶対パスまたは本スクリプトからの相対パスで指定。

## 補足
　スクリプトを実行すると、$OutputDir/keys下にサーバー・各クライアントの秘密鍵・公開鍵・事前共有鍵を記述したファイルが生成され、その後にそれらのファイルを読んで$OutputDirに設定ファイルが作られるという2段階のプロセスが発生します。スクリプトの実行が2度目以降の場合、1段階目で作られたファイルが残っている場合は再利用され、存在しないもののみ生成されます。そのため、鍵を作り直す場合はkeys以下のファイルを削除してから実行してください。鍵はそのままでその他の設定を変更したい場合は削除せずに実行してください。 例えばサーバーの設定のみを修正するときには、keys以下のファイルをそのままにしておくことにより、クライアント全ての設定ファイルをインストールし直すようなことが避けられます。生成したIPv6のプレフィックスもサーバーの鍵ファイル(keys/server.txt)に記述されるため、変更する場合はkeys/server.txtを削除してください。

　仮想インターフェイスのアドレスは、クライアント番号を4桁として、サーバーが192.168.100.1/16(IPv6 $IPv6Prefix::a000/96)、クライアントが192.168.「クライアント番号の上2桁」.「クライアント番号の下2桁」(IPv6 $IPv6Prefix:1::クライアント番号)に設定されます。クライアント数が9999以下ならば問題は起きないはずですが、別のアドレスを使用する場合はServer(Client)WgAddressを修正してください。その際は、Server(client)AlllowedIPsの修正も必要になるかと思われます。

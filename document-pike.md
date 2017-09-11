---
documentclass: ltjsarticle
title: OpenStack 構築手順書 Pike版
date: 0.9.2 (2017/09/11)
author: 日本仮想化技術株式会社
toc: yes
output:
  pdf_document:
    latex_engine: lualatex
    keep_tex: true
header-includes:
  - \usepackage[margin=1in]{geometry}
---

\clearpage

変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.9.0-1|2017/09/07|Pike版 初版|
|0.9.0-2|2017/09/08|要件などについて加筆|
|0.9.1|2017/09/08|表記揺れや誤記の修正。DashboardのUI変更への対応|
|0.9.2|2017/09/11|補足編として他のディストリビューションとの違いについて追加|


```
筆者注:
このドキュメントに対する提案や誤りの指摘はIssue登録か、
日本仮想化技術までメールにてお願いします。
https://github.com/virtualtech/openstack-pike-docs/issues
```


\clearpage


# OpenStack 構築構築

本章は、「[OpenStack Foundation](https://openstack.org)」が公開している公式ドキュメント「[OpenStack Installation Tutorial](https://docs.openstack.org/)」の内容から、「Keystone,Glance,Nova,Neutron,Horizon」までの構築手順をベースに加筆したものです。
OpenStackをUbuntu Server 16.04ベースで構築する手順を解説しています。

Canonical社が提供するCloud Archiveリポジトリーのパッケージを使って、OpenStackの最新版Pikeを導入しましょう。

Pikeでも旧来のバージョンにあった設定がいくつか削除されるなど、重要な変更が存在します。「[リリースノート](https://releases.openstack.org/pike/)」を構築を始める前にご覧ください。

\clearpage

## 構築する環境について

### 環境構築に使用するOS

本書はCanonicalのUbuntu ServerとCloud Archiveリポジトリーのパッケージを使って、OpenStack Pikeを構築する手順を解説したものです。

OpenStack PikeをUbuntuで構築するには、バージョン16.04(Xenial)が必要です。

本書は4.4.0-93以降のバージョンのカーネルで動作するUbuntu Server 16.04の最新版を使ってOpenStack Pikeを構築します。インストールイメージは以下からダウンロードできます。

- <http://archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/current/images/netboot/mini.iso>


```
筆者注:
もしここで想定するUbuntuやカーネルバージョン以外で何らかの事象が発生した場合も、
以下までご報告ください。動作可否情報をGithubで共有できればと思います。
https://github.com/virtualtech/openstack-pike-docs/issues
Ubuntu LTSとカーネルの関係については次のWikiをご覧ください。
https://wiki.ubuntu.com/Kernel/LTSEnablementStack
```


\clearpage

### 作成するサーバー（ノード）

本書はOpenStack環境をController,Computeの2台のサーバー上に構築することを想定しています。

| コントローラー   | コンピュート
| -------------- | --------------
| RabbitMQ       | Linux KVM
| NTP            | Nova Compute
| MariaDB        | Linux Bridge Agent
| Keystone       
| Glance
| Nova
| Neutron Server
| Linux Bridge Agent
| L3 Agent
| DHCP Agent
| Metadata Agent


### ノードの性能要件

最低限、以下のスペックを満たす必要があります。これはOpenStack Pike環境を作って、4個程度の[CirrOS](http://download.cirros-cloud.net/)、もしくは2個程度の[Ubuntu](http://cloud-images.ubuntu.com/releases/16.04/)インスタンスを起動することを想定しています。要件にあったスペックをご用意ください。

|内容   |コントローラー|コンピュート|
|-----|-------|------|
|CPU  |3      |4     |
|メモリー |6GB    |4GB   |
|ストレージ|30GB   |30GB  |

\clearpage

### ネットワークセグメントの設定

IPアドレスは以下の構成で構築されている前提で解説します。

|各種設定|ネットワーク|
|:---|:---|
|ネットワーク|10.0.0.0/24|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

### 各ノードのネットワーク設定

各ノードのネットワーク設定は以下の通りです。
Ubuntu 16.04ではNICのデバイス名の命名規則が変わりました。
物理サーバー上のNICはハードウェアとの接続によってensXやenoX、enpXsYのような命名規則になっています。なお、仮想マシンやコンテナーではethXやensXと認識されるようです。

本例ではens3として認識されているのを想定していますので、実際の環境に合わせて適宜読み替えてください。

+ controllerノード

|インターフェース|ens3|
|:---|:---|
|IPアドレス|10.0.0.111|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

+ computeノード

|インターフェース|ens3|
|:---|:---|
|IPアドレス|10.0.0.112|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

\clearpage

## Ubuntu Serverのインストール

### インストール

2台のサーバーに対し、Ubuntu Serverをインストールします。要点は以下の通りです。

+ 優先ネットワークインターフェースをens3(最初の方のNIC)に指定
    + インターネットへ接続するインターフェースはens3を使用するため、インストール中はens3を優先ネットワークとして指定
+ OSは最小インストール
    + パッケージ選択ではOpenSSH serverとBasic Ubuntu serverを追加

パスワードを入力後、Weak passwordという警告が出た場合はYesを選択するか、警告が出ないようなより複雑なパスワードを設定してください。

```
筆者注
Ubuntuインストーラーは基本的にインストール時にインターネット接続が必要です。

Ubuntuインストール時に選択した言語がインストール後も使われます。

Ubuntu Serverで日本語の言語を設定した場合、標準出力や標準エラー出力が
文字化けするため、言語は英語を設定されることを推奨します。
```

\clearpage

### プロキシーの設定

外部ネットワークとの接続にプロキシーの設定が必要な場合は、aptコマンドを使ってパッケージの照会やダウンロードを行うために次のような設定をする必要があります。

+ システムのプロキシー設定

```
# vi /etc/environment
http_proxy="http://proxy.example.com:8080/"
https_proxy="https://proxy.example.com:8080/"
no_proxy=localhost,controller,compute
```

+ APTのプロキシー設定

```
# vi /etc/apt/apt.conf
Acquire::http::proxy "http://proxy.example.com:8080/";
Acquire::https::proxy "https://proxy.example.com:8080/";
```

より詳細な情報は下記のサイトの情報を確認ください。

- <https://help.ubuntu.com/community/AptGet/Howto>
- <http://gihyo.jp/admin/serial/01/ubuntu-recipe/0331>

\clearpage

### Ubuntu Serverへのログインとroot権限

Ubuntuはデフォルト設定でrootユーザーの利用を許可していないため、root権限が必要となる作業は以下のように行ってください。

+ rootユーザーで直接ログインできないので、インストール時に作成したアカウントでログインする。
+ root権限が必要な作業を実行する場合には、sudoを実行したいコマンドの前につけて実行する。
+ root権限で連続して作業したい場合には、sudo -iコマンドでシェルを起動する。

## 特定のサービスが正常に動いていないと思われる場合

OpenStackのサービスが正常に動いていないと思われる場合、まずログを確認します。
正常に動いていないサービスを特定したら、そのサービスの設定ファイルに`debug=true`を記述してサービスを再起動します。
`debug=true`は`[DEFAULT]`の中に記述すると有効になります。
ログを確認し、出力されたエラーメッセージから既知の問題であって対処方法が公開されていないか確認します。

## 設定ファイル等の記述について

+ 設定ファイルの変更前と変更後に、設定ファイルに書かれている現在の設定を確認しましょう。次のように実行すると、コメントアウトされたものが除かれて表示されます。

```
# less <設定ファイル> | egrep -v "^\s*$|^\s*#"
```

+ 設定ファイルは特別な記述が無い限り、必要な設定を抜粋したものです。
+ 特に変更の必要がない設定項目は省略されています。
+ [見出し]が付いている場合、その見出しから次の見出しまでの間に設定を記述します。
+ コメントアウトされていない設定項目が存在する場合には、値を変更してください。多くの設定項目は記述が存在しているため、エディタの検索機能で検索することをお勧めします。
+ 特定のホストでコマンドを実行する場合はコマンドの冒頭にホスト名を記述しています。


【設定ファイルの記述例】

```
controller# vi /etc/glance/glance-api.conf

[DEFAULT]
debug=true    ← コメントをはずす

[database]
#connection = sqlite:////var/lib/glance/glance.sqlite   ← コメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記

[keystone_authtoken] ← 見出し
#auth_host = 127.0.0.1 ← コメントアウト
auth_host = controller ← 追記

auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0 ← 追記
admin_tenant_name = service ← 変更
admin_user = glance ← 変更
admin_password = password ← 変更
```

\clearpage


# OpenStackインストール前の設定

OpenStackパッケージのインストール前に各々のノードで以下の設定を行います。

+ ネットワークデバイスの設定
+ ホスト名と静的な名前解決の設定
+ リポジトリーの設定とパッケージの更新
+ Chronyサーバーのインストール（コントローラーノードのみ）
+ Chronyクライアントのインストール
+ Python用MySQL/MariaDBクライアントのインストール
+ MariaDBのインストール（コントローラーノードのみ）
+ RabbitMQのインストール（コントローラーノードのみ）
+ 環境変数設定ファイルの作成（コントローラーノードのみ）
+ Memcachedのインストールと設定（コントローラーノードのみ）

\clearpage

## ネットワークデバイスの設定

各ノードの/etc/network/interfacesを編集し、IPアドレスの設定を行います。

+ controllerノードのIPアドレスの設定

```
controller# vi /etc/network/interfaces
auto ens3
iface ens3 inet static
      address 10.0.0.111
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```

+ computeノードのIPアドレスの設定

```
compute# vi /etc/network/interfaces
auto ens3
iface ens3 inet static
      address 10.0.0.112
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```

+ ネットワークの設定を反映

各ノードで変更した設定を反映させるため、ホストを再起動します。

```
$ sudo reboot
```

\clearpage

## ホスト名と静的な名前解決の設定

ホスト名でノードにアクセスするにはDNSサーバーで名前解決する方法やhostsファイルに書く方法が挙げられます。
本書では各ノードの/etc/hostsに各ノードのIPアドレスとホスト名を記述してhostsファイルを使って名前解決します。127.0.1.1の行はコメントアウトします。

+ 各ノードのホスト名の設定

各ノードのホスト名をhostnamectlコマンドを使って設定します。反映させるためには一度ログインしなおす必要があります。

（例）controllerノードの場合

```
controller# hostnamectl set-hostname controller
controller# cat /etc/hostname
controller
```

+ 各ノードの/etc/hostsの設定

すべてのノードで127.0.1.1の行をコメントアウトします。
またホスト名で名前引きできるように設定します。

127.0.1.1の行はUbuntuの場合、デフォルトで設定されています。
Memcachedをセットアップするノードでこの設定がされたままだと正常に動作しませんので注意してください。

（例）controllerノードの場合

```
controller# vi /etc/hosts
127.0.0.1 localhost
#127.0.1.1 controller  ← コメントアウト
10.0.0.111 controller
10.0.0.112 compute
```

\clearpage

## リポジトリーの設定とパッケージの更新

OpenStack環境をセットアップする全てのノードで以下のコマンドを実行し、Pike向けUbuntu Cloud Archiveリポジトリーを登録します。

```
# add-apt-repository cloud-archive:pike
 Ubuntu Cloud Archive for OpenStack Pike
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it  ← ここでEnterキーを押下
...
Importing ubuntu-cloud.archive.canonical.com keyring
OK
Processing ubuntu-cloud.archive.canonical.com removal keyring
gpg: /etc/apt/trustdb.gpg: trustdb created
OK
```

各ノードのシステムをアップデートします。Ubuntuではパッケージのインストールやアップデートの際にまず`apt update`を実行してリポジトリー情報の更新が必要です。そのあと`apt upgrade`でアップグレードを行います。カーネルの更新があった場合は再起動してください。

なお、`apt update`は頻繁に実行する必要はありません。日をまたいで作業する際や、コマンドを実行しない場合にパッケージ更新やパッケージのインストールでエラーが出る場合は実行してください。以降の手順では`apt update`を省略します。

```
# apt update -q && apt upgrade
```

\clearpage

## OpenStackクライアントのインストール

OpenStackクライアントをインストールします。依存するパッケージは全てインストールします。

```
controller# apt install python-openstackclient
```

## 時刻同期サーバーのインストールと設定

+ 時刻同期サーバーChronyの設定

aptコマンドで各ノードの時刻を同期するためにChronyをインストールします。

```
# apt install chrony
```

+ controllerノードの時刻同期サーバーの設定

controllerノードで公開NTPサーバーと同期するNTPサーバーを構築します。
適切な公開NTPサーバー(ex.ntp.nict.jp etc..)をserverもしくはpoolで指定します。
ネットワーク内にNTPサーバーがある場合はそのサーバーを指定します。

```
controller# vi /etc/chrony/chrony.conf
...
server NTP_SERVER iburst    ← 同期するNTPサーバーを指定
...
allow 10.0.0.0/24    ←接続を許可するネットワーク
```

設定を変更した場合はNTPサービスを再起動します。

```
controller# service chrony restart
```

\clearpage

+ その他ノードの時刻同期サーバーの設定

computeノードでcontrollerノードと同期するNTPサーバーを構築します。

```
compute# vi /etc/chrony/chrony.conf

#pool 2.debian.pool.ntp.org offline iburst  ← デフォルト設定はコメントアウト
server controller iburst
```

設定を適用するため、NTPサービスを再起動します。

```
compute# service chrony restart
```

### NTPサーバーの動作確認

構築した環境でコマンドを実行して、各NTPサーバーが同期していることを確認します。

- 公開NTPサーバーと同期しているcontrollerノード

```
controller# chronyc sources
chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* chobi.paina.jp                2   8    17     1    +47us[ -312us] +/-   13ms
^- v157-7-235-92.z1d6.static     2   8    17     1  +1235us[+1235us] +/-   45ms
^- edm.butyshop.com              3   8    17     0  -2483us[-2483us] +/-   82ms
^- y.ns.gin.ntt.net              2   8    17     0  +1275us[+1275us] +/-   35ms
```

- controllerノードと同期しているその他ノード

```
compute# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* controller                3   6    77    25   -509us[-1484us] +/-   13ms
```

\clearpage

## MariaDBのインストール

データベースサーバーのMariaDBをインストールします。

### パッケージのインストール

aptコマンドでmariadb-serverパッケージをインストールします。

```
controller# apt install mariadb-server python-pymysql
```

### MariaDBの設定を変更

OpenStack用のMariaDB設定ファイル /etc/mysql/mariadb.conf.d/99-openstack.cnfを作成し、以下を設定します。

別のノードからMariaDBへアクセスできるようにするためバインドアドレスを変更します。加えて使用する文字コードをutf8に変更します。また、デフォルトの接続数ではOpenStackで利用するには適さないため、接続数を4096まで増やします。なお、この設定は規模に応じて適切な設定を行ってください。

※文字コードをutf8に変更しないとOpenStackモジュールとデータベース間の通信でエラーが発生します。

```
controller# vi /etc/mysql/mariadb.conf.d/99-openstack.cnf

[mysqld]
bind-address = 10.0.0.111            ← controllerのIPアドレス
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

\clearpage

### MariaDBサービスの再起動

変更した設定を反映させるためMariaDBのサービスを再起動します。

```
controller# service mysql restart
```

### MariaDBのセットアップ

MariaDBデータベースのセキュリティーを設定するにはmysql_secure_installationコマンドを実行します。
このコマンドを利用してMariaDBのrootユーザーのパスワードの設定やデフォルトで作られるユーザーやデータベースの削除なども行えます。

```
controller# mysql_secure_installation
```

\clearpage

## RabbitMQのインストール

OpenStackは、オペレーションやステータス情報を各サービス間で連携するためにメッセージブローカーを使用しています。OpenStackではRabbitMQ、ZeroMQなど複数のメッセージブローカーサービスに対応しています。
本書ではRabbitMQをインストールする例を説明します。

### パッケージのインストール

aptコマンドでrabbitmq-serverパッケージをインストールします。

```
controller# apt install rabbitmq-server
```

### openstackユーザーの作成とパーミッションの設定

RabbitMQにアクセスするためのユーザーとしてopenstackユーザーを作成し、必要なパーミッションを設定します。
以下コマンドはRabbitMQのopenstackユーザーのパスワードをpasswordに設定する例です。

```
controller# rabbitmqctl add_user openstack password
controller# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### RabbitMQの待ち受けに関する設定

controllerノードでRabbitMQの待ち受けに関する設定を行います。NODE_IP_ADDRESSにBind IPアドレスを設定します。
設定変更後はrabbitmq-serverサービスを再起動します。

```
controller# vi /etc/rabbitmq/rabbitmq-env.conf
NODE_IP_ADDRESS=10.0.0.111   ← RabbitMQサーバーノード(本例はcontroller)を指定

controller# service rabbitmq-server restart
```

\clearpage

### RabbitMQサービスのログを確認

+ ログの確認

メッセージブローカーサービスが正常に動いていないと、OpenStackの各コンポーネントは正常に動きません。RabbitMQサービスの再起動と動作確認を行い、確実に動作していることを確認します。

```
controller# tailf /var/log/rabbitmq/rabbit@controller.log
...
=INFO REPORT==== 4-Sep-2017::17:14:09 ===
msg_store_persistent: using rabbit_msg_store_ets_index to provide index

=INFO REPORT==== 4-Sep-2017::17:14:09 ===
started TCP Listener on 172.17.14.111:5672  ← 待受けIPアドレスとポートを確認

=INFO REPORT==== 4-Sep-2017::17:14:09 ===
Server startup complete; 0 plugins started.
```

\clearpage


## Memcachedのインストールと設定

認証機構のトークンをキャッシュするための用途でMemcachedをインストールします。

```
controller# apt install memcached python-memcache
```

設定ファイル /etc/memcached.conf を変更します。
既存の行 -l 127.0.0.1 をcontrollerノードのIPアドレスに変更します。

```
controller# vi /etc/memcached.conf
-l 10.0.0.111    ← controllerのIPアドレス
```

memcachedサービスを再起動します。

```
controller# service memcached restart
```

\clearpage


# Keystoneのインストールと設定（コントローラーノード）

各サービス間の連携時に使用する認証サービスKeystoneのインストールと設定を行います。

## データベースを作成

MariaDBにKeystoneで使用するデータベースを作成しアクセス権を付与します。

```
# mysql   ← MariaDBにログイン
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'password';

MariaDB [(none)]> quit
```

## データベースの確認

MariaDBにkeystoneユーザーでログインしデータベースの閲覧が可能であることを確認します。

```
controller# mysql -u keystone -p
Enter password:  ← MariaDBのkeystoneパスワードpasswordを入力
...
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
```

\clearpage

## パッケージのインストール

aptコマンドでkeystoneおよび必要なパッケージをインストールします。

```
controller# apt install keystone apache2 libapache2-mod-wsgi
```

## Keystoneの設定変更

keystoneの設定ファイルを変更します。

```
controller# vi /etc/keystone/keystone.conf
...
[database]
#connection = sqlite:////var/lib/keystone/keystone.db   ← 既存設定をコメントアウト
connection = mysql+pymysql://keystone:password@controller/keystone   ← 追記
...
[token]
...
provider = fernet       ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/keystone/keystone.conf | egrep -v "^\s*$|^\s*#"
```

### データベースに展開

```
controller# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

\clearpage

### Fernetキーの初期化

keystone-manageコマンドを実行してFernetキーを初期化します。

```
controller# keystone-manage fernet_setup --keystone-user keystone \
 --keystone-group keystone
controller# keystone-manage credential_setup --keystone-user keystone \
 --keystone-group keystone
```

### Identityサービスのデプロイ

次のコマンドを実行してIdentityサービスをデプロイします。`--bootstrap-password`で、Keystoneのadminユーザーに設定するパスワードを指定します。

```
controller# keystone-manage bootstrap --bootstrap-password password \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:35357/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### Apache Webサーバーの設定

+ apache2.confに項目ServerNameを追加して、controllerノードのホスト名を設定します。

```
controller# vi /etc/apache2/apache2.conf
...
# Global configuration
#
ServerName controller
...
```

## サービスの再起動と不要DBの削除

+ Apache Webサーバーを再起動します。

```
controller# service apache2 restart
```

+ パッケージのインストール時に作成される不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/keystone/keystone.db
```

\clearpage

## サービスとAPIエンドポイントの作成

以下のコマンドでサービスとAPIエンドポイントを設定します。

+ 環境変数の設定

```
# vi tmp-openrc.sh
```

+ スクリプトを作成

```
#!/bin/bash

export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

前の手順でKeystoneのadminユーザーに設定したパスワードを`OS_PASSWORD`に指定します。

+ スクリプトをシステムに適用

```
# source tmp-openrc.sh
```

+ サービスの作成

```
controller# openstack project create --domain default \
  --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 24ac7f19cd944f4cba1d77469b2a73ed |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```

\clearpage

## プロジェクトとユーザー、ロールの作成

+ demoプロジェクトの作成

```
controller# openstack project create --domain default \
  --description "Demo Project" demo

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
```

+ demoユーザーの作成

```
controller# openstack user create --domain default \
  --password-prompt demo

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aeda23aa78f44e859900e22c24817832 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

\clearpage

+ userロールの作成

```
controller# openstack role create user

+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 997ce8d05fc143ac97d83fdfb5998552 |
| name      | user                             |
+-----------+----------------------------------+
```

+ userロールをdemoプロジェクトとdemoユーザーに追加

```
controller# openstack role add --project demo --user demo user
```

このコマンドを実行しても特に何も出力されません。

\clearpage


## Keystoneの動作確認

他のサービスをインストールする前にKeystoneが正しく構築、設定されたか動作を検証します。

+ 一時的に設定した環境変数を解除します。

```
controller# unset OS_AUTH_URL OS_PASSWORD
```

+ セキュリティーを確保するため、一時認証トークンメカニズムを無効化します。
    + keystone-paste.iniを開き、[pipeline:public_api]と[pipeline:admin_api]と[pipeline:api_v3]セクション（訳者注:..のpipeline行) に`admin_token_auth`があれば、削除します。

```
controller# grep admin_token_auth /etc/keystone/keystone-paste.ini
(設定ファイルを確認)

controller# vi /etc/keystone/keystone-paste.ini
(設定ファイルにadmin_token_authがあれば取り除く)
```

\clearpage

動作確認のため、adminおよびdemoテナントに対し認証トークンを要求します。
admin、demoユーザーのパスワードを入力します。

+ adminユーザーとdemoユーザーとして認証トークンを要求します。

```
controller# openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
Password:    ← adminユーザーにパスワードを入力

controller# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue
Password:    ← demoユーザーにパスワードを入力
```

\clearpage


## OpenStackクライアントのRCスクリプトの作成

コマンドラインでKeystone認証を通すために、次のようなRCスクリプトを作成します。
adminユーザー用の`admin-openrc`と、demoユーザー用の`demo-openrc`を作成します。

```
controller# cat admin-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

controller# cat demo-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

sourceコマンドで各RCスクリプトを読み込んだ後、結果が出力されることを確認します。

```
# openstack token issue
```

\clearpage


# Glanceのインストールと設定

## データベースを作成

MariaDBにglanceで使用するデータベースを作成し、アクセス権を付与します。

```
controller# mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
 IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'password';

MariaDB [(none)]> quit
```

## データベースの確認

MariaDBにglanceユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u glance -p
Enter password: ← MariaDBのglanceパスワードpasswordを入力
...
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
```

\clearpage

## ユーザー、サービス、APIエンドポイントの作成

以下のコマンドで認証情報を読み込み、ImageサービスとAPIエンドポイントを設定します。

### 環境変数ファイルの読み込み

admin-openrcを読み込みます。

```
controller# source admin-openrc
```

### glanceユーザーの作成

```
controller# openstack user create --domain default --password-prompt glance
User Password: password  ← glanceユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3051a763c5754b77a51b67bf8c2da06b |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

### adminロールをglanceユーザーとserviceプロジェクトに追加

```
controller# openstack role add --project service --user glance admin
```
\clearpage

### glanceサービスの作成

```
controller# openstack service create --name glance \
--description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

### APIエンドポイントの作成

```
controller# openstack endpoint create --region RegionOne \
  image public http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image internal http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

## Glanceのインストール

aptコマンドでglanceパッケージをインストールします。

```
controller# apt install glance
```

\clearpage

## Glanceの設定変更

Glance(APIとレジストリー)の設定を行います。

```
controller# vi /etc/glance/glance-api.conf
...

[database]
#sqlite_db = /var/lib/glance/glance.sqlite      ← 既存設定があればコメントアウト
connection = mysql+pymysql://glance:password@controller/glance   ← 追記
...
[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/  ← 変更
...
[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password  ← glanceユーザーのパスワード
...
[paste_deploy]
...
flavor = keystone          ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-api.conf | egrep -v "^\s*$|^\s*#"
```

\clearpage

```
controller# vi /etc/glance/glance-registry.conf

[DEFAULT]
...
[database]
#sqlite_db = /var/lib/glance/glance.sqlite      ← 既存設定があればコメントアウト
connection = mysql+pymysql://glance:password@controller/glance  ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = password      ← glanceユーザーのパスワード

...
[paste_deploy]
flavor = keystone                ← 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-registry.conf | egrep -v "^\s*$|^\s*#"
```

### データベースに展開

次のコマンドでglanceデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "glance-manage db_sync" glance
...
Upgraded database to: pike01, current revision(s): pike01
```

インストールした時期によって異なるメッセージが表示されることがあるかもしれませんが、
`Upgraded database to: pike01`のようなメッセージが出力され、エラー出力がないことを確認します。

\clearpage

### Glanceサービスの再起動

設定を反映させるためGlanceサービスを再起動します。

```
controller# service glance-registry restart
controller# service glance-api restart
```

### ログの確認と使用しないデータベースファイルの削除

サービスの再起動後、ログを参照しGlanceの各サービスでエラーが起きていないことを確認します。

```
controller# tailf /var/log/glance/glance-api.log
controller# tailf /var/log/glance/glance-registry.log
```

## イメージの取得と登録

Glanceへインスタンス用の仮想マシンイメージを登録します。ここでは、OpenStackのテスト環境に役立つ軽量なLinuxイメージ CirrOS を登録します。

### イメージの取得

CirrOSのWebサイトより仮想マシンイメージをダウンロードします。

```
controller# wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

### イメージを登録

ダウンロードした仮想マシンイメージをGlanceに登録します。

```
controller# source admin-openrc
controller# openstack image create "cirros" \
  --file cirros-0.3.5-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

  +------------------+------------------------------------------------------+
  | Field            | Value                                                |
  +------------------+------------------------------------------------------+
  | checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |
  | container_format | bare                                                 |
  | created_at       | 2017-09-05T04:34:49Z                                 |
  | disk_format      | qcow2                                                |
  | file             | /v2/images/30c9a014-b94a-4368-a883-ed738818f955/file |
  | id               | 30c9a014-b94a-4368-a883-ed738818f955                 |
  | min_disk         | 0                                                    |
  | min_ram          | 0                                                    |
  | name             | cirros                                               |
  | owner            | f44977aee5b0490f9cd4de231b3f569e                     |
  | protected        | False                                                |
  | schema           | /v2/schemas/image                                    |
  | size             | 13267968                                             |
  | status           | active                                               |
  | tags             |                                                      |
  | updated_at       | 2017-09-05T04:34:50Z                                 |
  | virtual_size     | None                                                 |
  | visibility       | public                                               |
  +------------------+------------------------------------------------------+
```

### イメージの登録を確認

仮想マシンイメージが正しく登録されたか確認します。

```
controller# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 30c9a014-b94a-4368-a883-ed738818f955 | cirros | active |
+--------------------------------------+--------+--------+
```

\clearpage


# Novaのインストールと設定（コントローラーノード）

## データベースを作成

MariaDBにデータベースnovaを作成します。

```
controller# mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'password';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'password';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'password';

MariaDB [(none)]> quit
```

## データベースの確認

MariaDBにnovaユーザーでログインし、データベースの閲覧が可能であることを確認します。

```
controller# mysql -u nova -p
Enter password: ← MariaDBのnovaパスワードpasswordを入力
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
| nova_api           |
| nova_cell0         |
+--------------------+
```

\clearpage

## ユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、サービスとAPIエンドポイントを設定します。

### 環境変数ファイルの読み込み

```
controller# source admin-openrc
```

### novaユーザーの作成

```
controller# openstack user create --domain default --password-prompt nova
User Password: password  ← novaユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8a7dbf5279404537b1c7b86c033620fe |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

### novaユーザーをadminロールに追加

```
controller# openstack role add --project service --user nova admin
```

\clearpage

### novaサービスの作成

```
controller# openstack service create --name nova \
  --description "OpenStack Compute" compute

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 060d59eac51b4594815603d75a00aba2 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

### ComputeサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
controller# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
controller# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
```

### Placementサービスのユーザーを作成

```
controller# openstack user create --domain default --password-prompt placement
User Password: password  ← placementユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fa742015a6494a949f67629884fc7ec8 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

\clearpage

### Placementユーザーをadminロールに追加

```
controller# openstack role add --project service --user placement admin
```

### Placementサービスの作成

```
controller# openstack service create --name placement \
  --description "Placement API" placement
```

### PlacementサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  placement public http://controller:8778
controller# openstack endpoint create --region RegionOne \
  placement internal http://controller:8778
controller# openstack endpoint create --region RegionOne \
  placement admin http://controller:8778
```

\clearpage


## パッケージのインストール

aptコマンドでNova関連のパッケージをインストールします。

```
controller# apt install nova-api nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler nova-placement-api
```

## Novaの設定変更

nova.confの設定を行います。
すでにいくつかの設定は行われています。各セクションから設定項目を探して同じように設定するか、項目が見つからない場合は設定を追加してください。言及していない設定はそのままで構いません。viで`/[DEFAULT`のようにセクション名の最後の括弧をのぞいて検索すると、項目を見つけられやすいと思います。

`[database]`のconnectionと、`[placement]`のos_region_nameが定義済みになっていました。別の設定をする場合は注意しましょう。引き続き`[DEFAULT]`セクションにlock_pathが定義されていますので、コメントアウトするようにしましょう。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
...
#lock_path = /var/lock/nova  ← コメントアウト
my_ip = 10.0.0.111
transport_url = rabbit://openstack:password@controller
use_neutron = True
vif_plugging_is_fatal = True
vif_plugging_timeout = 300

[api]
...
auth_strategy = keystone

[api_database]
#connection = sqlite:////var/lib/nova/nova_api.sqlite  ← コメントアウト
...
connection = mysql+pymysql://nova:password@controller/nova_api

[database]
#connection = sqlite:////var/lib/nova/nova.sqlite  ← コメントアウト
connection = mysql+pymysql://nova:password@controller/nova

[glance]
...
api_servers = http://controller:9292
```

\clearpage

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[vnc]
enabled = true
...
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp

[placement]
#os_region_name = openstack    ← コメントアウト
...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:35357/v3
username = placement
password = password
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | egrep -v "^\s*$|^\s*#"
```

\clearpage

### データベースに展開

次のコマンドでnovaデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "nova-manage api_db sync" nova
controller# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
controller# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
controller# su -s /bin/sh -c "nova-manage db sync" nova
```

最後のコマンドは処理に少々、時間がかかります。


### Nova Cellの確認

```
controller# nova-manage cell_v2 list_cells
```

### Novaサービスの再起動

設定を反映させるため、Novaのサービスを再起動します。

```
controller# service nova-api restart
controller# service nova-consoleauth restart
controller# service nova-scheduler restart
controller# service nova-conductor restart
controller# service nova-novncproxy restart
```

### 不要なデータベースファイルの削除

データベースはMariaDBを使用するため、不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/nova/nova.sqlite
controller# rm /var/lib/nova/nova_api.sqlite
```

\clearpage


# Nova Computeのインストールと設定（コンピュートノード）

ここまでcontrollerノードの環境構築を行ってきましたが、ここでcomputeノードに切り替えて設定を行います。

## パッケージのインストール

aptコマンドを使って、nova-computeパッケージをインストールします。多くのパッケージをインストールするため、インストール完了まで少々時間を要します。

```
compute# apt install nova-compute
```

## Novaの設定を変更

nova.confの設定を行います。
すでにいくつかの設定は行われています。各セクションから設定項目を探して同じように設定するか、項目が見つからない場合は設定を追加してください。言及していない設定はそのままで構いません。viで`/[DEFAULT`のようにセクション名の最後の括弧をのぞいて検索すると、項目を見つけられやすいと思います。

computeノードではデータベースサーバーとの接続は不要なため、設定ファイルにデータベースの項目が設定されている場合はコメント化してください。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]
#log_dir = /var/log/nova   ← コメントアウト
#lock_path = /var/lock/nova   ← コメントアウト
...
transport_url = rabbit://openstack:password@controller
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
my_ip = 10.0.0.112

[api]
...
auth_strategy = keystone

[glance]
...
api_servers = http://controller:9292

[vnc]
...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

\clearpage

```
(→前のページからの続き)

[api_database]
#connection = sqlite:////var/lib/nova/nova_api.sqlite  ← コメントアウト

[database]
...
#connection = sqlite:////var/lib/nova/nova.sqlite  ← コメントアウト

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = password

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp

[placement]
# os_region_name = openstack  ← コメントアウト
...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:35357/v3
username = placement
password = password
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | egrep -v "^\s*$|^\s*#"
```

\clearpage

まず次のコマンドを実行し、computeノードでLinux KVMが動作可能であることを確認します。コマンド結果が1以上の場合は、CPU仮想化支援機能がサポートされています。
もしこのコマンド結果が0の場合は仮想化支援機能がサポートされていないか、設定が有効化されていないので、libvirtでKVMの代わりにQEMUを使用します。後述のnova-compute.confの設定で`virt_type = qemu`を設定します。

```
compute# cat /proc/cpuinfo |egrep 'vmx|svm'|wc -l
4
```

VMXもしくはSVM対応CPUの場合は「virt_type = kvm」と設定することにより、仮想化部分のパフォーマンスが向上します。先のコマンドの実行結果が0になる場合は「virt_type = qemu」を設定してください。

```
compute# vi /etc/nova/nova-compute.conf

[libvirt]
...
virt_type = kvm
```

## Novaコンピュートサービスの再起動

設定を反映させるため、nova-computeサービスを再起動します。

```
compute# service nova-compute restart
```

nova-computeサービスが起動しない場合は/var/log/nova/nova-compute.logを確認してください。Ocata以降ではPlacement APIの設定がnova.confに存在しないとnova-computeサービスを起動できませんので注意してください。

また、`AMQP server on controller:5672 is unreachable`のようなメッセージが出る場合はcontrollerのRabbitMQサービスが起動しているか、Bind IPアドレスやポートは適切かを確認します。

\clearpage

## コントローラーノードとの疎通確認

computeとcontrollerノードの疎通を確認します。操作するノードをcontrollerノードに切り替えて、各コマンドを実行してください。

```
controller# source admin-openrc
```

### Cellデータベースへのコンピュートノードの追加

```
controller# openstack compute service list --service nova-compute
+----+--------------+---------+------+---------+-------+----------------------------+
| ID | Binary       | Host    | Zone | Status  | State | Updated At                 |
+----+--------------+---------+------+---------+-------+----------------------------+
|  6 | nova-compute | compute | nova | enabled | up    | 2017-09-05T07:07:03.000000 |
+----+--------------+---------+------+---------+-------+----------------------------+

controller# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': 3da28a07-3e02-4b58-99bd-bc03a120b81c
Found 1 unmapped computes in cell: 3da28a07-3e02-4b58-99bd-bc03a120b81c
Checking host mapping for compute host 'compute': 6ac62de6-08bf-4bba-aa4e-f1027419bc59
Creating host mapping for compute host 'compute': 6ac62de6-08bf-4bba-aa4e-f1027419bc59
```

\clearpage

### Novaサービスの確認

Novaコンピュートサービスの状態を確認します。`openstack compute service list`コマンドで関連サービスのステータスを確認します。また`openstack catalog list`コマンドでIdentityサービスのAPIエンドポイントのリストを表示して、サービスとの接続を確認します。nova,placement,keystone,glanceサービスが表示され、それぞれpublic,admin,internalのエンドポイントが表示されていることを確認します。

また、イメージサービスから情報を引き出せるか確認するため、`openstack image list`を実行します。CellsとPlacement APIが正常に動作しているか確認するために`nova-status upgrade check`コマンドを実行します。

```
controller# nova-status upgrade check
+---------------------------+
| Upgrade Check Results     |
+---------------------------+
| Check: Cells v2           |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Placement API      |
| Result: Success           |
| Details: None             |
+---------------------------+
| Check: Resource Providers |
| Result: Success           |
| Details: None             |
+---------------------------+
```

\clearpage


# Neutronのインストール・設定（コントローラーノード）

## データベースを作成

MariaDBにデータベースneutronを作成し、アクセス権を付与します。

```
controller# mysql
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'password';

MariaDB [(none)]> quit
```

## データベースの確認

MariaDBにneutronユーザーでログインし、データベースの閲覧が可能か確認します。

```
controller# mysql -u neutron -p
Enter password: ←MariaDBのneutronパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
```

※ユーザーneutronでログイン可能でデータベースが閲覧可能なら問題ありません。

\clearpage

## neutronユーザーとサービス、APIエンドポイントの作成

以下コマンドで認証情報を読み込んだあと、neutronサービスの作成とAPIエンドポイントを設定します。

### 環境変数ファイルの読み込み

```
controller# source admin-openrc
```

### neutronユーザーの作成

```
controller# openstack user create --domain default --password-prompt neutron
User Password: password  #neutronユーザーのパスワードを設定(本書はpasswordを設定)
Repeat User Password: password
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | fdb0f541e28141719b6a43c8944bf1fb |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

### neutronユーザーをadminロールに追加

```
controller# openstack role add --project service --user neutron admin
```

\clearpage

### neutronサービスの作成

```
controller# openstack service create --name neutron \
  --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | f71529314dab4a4d8eca427e701d209e |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

### neutronサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  network public http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network internal http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

## パッケージのインストール

本書ではネットワークの構成は公式マニュアルの「[Networking Option 2: Self-service networks](https://docs.openstack.org/neutron/pike/install/controller-install-option2-ubuntu.html)」の方法で構築する例を示します。


```
controller# apt install neutron-server neutron-plugin-ml2 \
 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
 neutron-metadata-agent
```

\clearpage

## Neutronコンポーネントの設定

### Neutronサーバーの設定

controllerノードのneutron.confの設定を行います。
すでにいくつかの設定は行われています。各セクションから設定項目を探して同じように設定するか、項目が見つからない場合は設定を追加してください。言及していない設定はそのままで構いません。viで`/[DEFAULT`のようにセクション名の最後の括弧をのぞいて検索すると、項目を見つけられやすいと思います。

```
controller# vi /etc/neutron/neutron.conf

[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
transport_url = rabbit://openstack:password@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite  ← コメントアウト
connection = mysql+pymysql://neutron:password@controller/neutron

(次のページに続きます→)
```

\clearpage

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[nova]
...
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password
```

[keystone_authtoken]セクションは追記した設定以外は取り除くかコメントアウトしてください。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/neutron.conf | egrep -v "^\s*$|^\s*#"
```

\clearpage


### ML2プラグインの設定

ml2_conf.iniの設定を行います。

```
controller# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_vxlan]
...
vni_ranges = 1:1000

[securitygroup]
...
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = True
enable_ipset = True
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/ml2_conf.ini | egrep -v "^\s*$|^\s*#"
```

\clearpage

### Linuxブリッジエージェントの設定

パブリックネットワークに接続している側のNICを指定します。本書ではens3を指定します。
今回の構成の場合は「flat_networksで指定した値:パブリックネットワークに接続している側のNIC」を指定する必要があるので、「provider:ens3」と設定します。

```
controller# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:ens3 ←追記
```

local_ipは、先にphysical_interface_mappingに設定したNIC側のIPアドレスを設定します。

```
[vxlan]
enable_vxlan = True         ← コメントをはずす
local_ip = 10.0.0.111       ← 変更
l2_population = True        ← 変更 ※
```

エージェントとセキュリティグループの設定を行います。

```
[securitygroup]
...
enable_security_group = True    ← コメントをはずす
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
↑ 追記
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | egrep -v "^\s*$|^\s*#"
```

\clearpage

#### ※ ML2プラグインのl2populationについて

OpenStack Mitaka以降のバージョンの公式手順書では、ML2プラグインのmechanism_driversとしてlinuxbridgeとl2populationが指定されています。l2populationが有効だとこれまでの動きと異なり、インスタンスが起動してもネットワークが有効化されるまで通信ができません。つまりNeutronネットワークを構築してルーターのパブリック側のIPアドレス宛にPingコマンドを実行しても疎通ができません。このネットワーク有効化の有無についてメッセージキューサービスが監視して制御しています。
従って、これまでのバージョン以上にメッセージキューサービス（本例や公式手順ではしばしばRabbitMQが使用されます）が確実に動作している必要があります。このような仕組みが導入されたのは、不要なパケットがネットワーク内に流れ続けないようにするためです。

ただし、弊社でESXi仮想マシン環境に構築したOpenStack環境においてl2populationが有効化されていると想定通り動かないという事象が発生することを確認してます。その他のハイパーバイザーでは確認していませんが、ネットワーク通信に支障が起きる場合はl2populationをオフに設定すると改善される場合があります。修正箇所は次の通りです。

+ controllerの/etc/neutron/plugins/ml2/ml2_conf.iniの設定変更

```
[ml2]
...
mechanism_drivers = linuxbridge,l2population
↓
mechanism_drivers = linuxbridge
```

+ controller & computeの/etc/neutron/plugins/ml2/linuxbridge_agent.iniの設定変更

```
[vxlan]
...
l2_population = true
↓
l2_population = false
```

設定変更後はl2populationの設定変更を反映させるため、controllerとcomputeノードのNeutron関連サービスを再起動するか、システムを再起動してください。

\clearpage

### Layer-3エージェントの設定


[DEFAULT]セクションで、利用するインターフェイスドライバーを指定します。Linuxブリッジを利用するため、次のように設定します。

```
controller# vi /etc/neutron/l3_agent.ini

[DEFAULT]
...
interface_driver = linuxbridge
```

### DHCPエージェントの設定

```
controller# vi /etc/neutron/dhcp_agent.ini

[DEFAULT]
...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
dnsmasq_config_file =/etc/neutron/dnsmasq-neutron.conf
```

### Dnsmasqの設定

Dnsmasqのログの設定とDHCPオプションの設定です。VXLANの追加ヘッダー対策のため、MTUを1454に設定するには次のようにDHCPオプションでパラメーターを渡すことができます。インスタンスが起動するときにインスタンスイメージにDHCPクライアントがある場合は全てのインスタンスでこのMTU値が反映されます。

```
controller# vi /etc/neutron/dnsmasq-neutron.conf
log-facility = /var/log/neutron/dnsmasq.log
log-dhcp
dhcp-option=26,1454
```

\clearpage

### Metadataエージェントの設定

インスタンスのメタデータサービスを提供するMetadataエージェントを設定します。

```
controller# vi /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 10.0.0.111 ← Metadataホストを指定
metadata_proxy_shared_secret = METADATA_SECRET

[cache]
memcache_servers = controller:11211
```

旧来の`nova_metadata_ip`ではなく、`nova_metadata_host`を利用します。

Metadataエージェントの`metadata_proxy_shared_secret`に指定する値と、次の手順でNovaに設定する`metadata_proxy_shared_secret`が同じ値になるように設定します。任意の値を設定すれば良いですが、思いつかない場合は次のように実行して生成した乱数を使うことも可能です。

```
controller# openssl rand -hex 10
```

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/neutron/metadata_agent.ini | egrep -v "^\s*$|^\s*#"
```

\clearpage

## Novaの設定を変更

Novaの設定ファイルの`[neutron]`セクションに、Neutronの設定を追記します。

```
controller# vi /etc/nova/nova.conf
...
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password   ←neutronユーザーのパスワード
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

METADATA_SECRETはMetadataエージェントで指定した値に置き換えます。

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | egrep -v "^\s*$|^\s*#"
```

## データベースに展開

コマンドを実行して、エラーが発生せずに完了することを確認します。

```
controller# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
...
INFO  [alembic.runtime.migration] Running upgrade f83a0b2964d0 -> fd38cd995cc0, change shared attribute for firewall resource
  OK    ← 最後にOKと出力されることを確認
```

\clearpage

## コントローラーノードのNeutronと関連サービスの再起動

設定を反映させるため、コントローラーノードの関連サービスを再起動します。
まずNova APIサービスを再起動します。

```
controller# service nova-api restart
```

次にNeutron関連サービスを再起動します。

```
controller# service neutron-server restart
controller# service neutron-linuxbridge-agent restart
controller# service neutron-dhcp-agent restart
controller# service neutron-metadata-agent restart

ontroller# service neutron-l3-agent restart
```

## ログの確認

ログを確認して、エラーが出力されていないことを確認します。

```
controller# tailf /var/log/nova/nova-api.log
controller# tailf /var/log/neutron/neutron-server.log
controller# tailf /var/log/neutron/neutron-metadata-agent.log
controller# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

## 使用しないデータベースファイルを削除

```
controller# rm /var/lib/neutron/neutron.sqlite
```

\clearpage

# Neutronのインストール・設定（コンピュートノード）

次にコンピュートノードの設定を行います。

## パッケージのインストール

```
compute# apt install neutron-linuxbridge-agent
```

## 設定の変更

### Neutronの設定

neutron.confの設定を行います。
すでにいくつかの設定は行われているので、各セクションに同じように設定がされているか確認し、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
compute# vi /etc/neutron/neutron.conf

[DEFAULT]
...
auth_strategy = keystone
transport_url = rabbit://openstack:password@controller

(次のページに続きます→)
```

\clearpage

```
(→前のページからの続き)

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password

[database]
...
#connection = sqlite:////var/lib/neutron/neutron.sqlite ←コメントアウト
```

本書の構成では、コンピュートノードのNeutron.confにはデータベースの指定は不要です。
データベースの指定がデフォルトで存在しています。この設定はコメントアウトしてもしなくても構いません。
次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/neutron.conf | egrep -v "^\s*$|^\s*#"
```

\clearpage

### Linuxブリッジエージェントの設定

linuxbridge_agent.iniの設定を行います。
すでにいくつかの設定は行われているので、各セクションに同じように設定がされているか確認し、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

physical_interface_mappingsにはパブリック側のネットワークに接続しているインターフェイスを指定します。本書ではens3を指定します。
local_ipにはパブリック側に接続しているNICに設定しているIPアドレスを指定します。

```
compute# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:ens3

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = True
local_ip = 10.0.0.112
l2_population = True  (※1)
```

※1 ML2プラグインの設定 /etc/neutron/plugins/ml2/ml2_conf.iniでl2populationを使用しない場合はl2_population = False を設定し無効化します。

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | egrep -v "^\s*$|^\s*#"
```

\clearpage

## コンピュートノードのネットワーク設定

Novaの設定ファイルの内容をNeutronを利用するように変更します。

```
compute# vi /etc/nova/nova.conf
...
[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password  ←neutronユーザーのパスワード
```

次のコマンドで正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | egrep -v "^\s*$|^\s*#"
```

## コンピュートノードのNeutronと関連サービスを再起動

ネットワーク設定を反映させるため、コンピュートノードのNeutronと関連のサービスを再起動します。

```
compute# service nova-compute restart
compute# service neutron-linuxbridge-agent restart
```

\clearpage

## ログの確認

エラーが出ていないかログを確認します。

```
compute# tailf /var/log/nova/nova-compute.log
compute# tailf /var/log/neutron/neutron-linuxbridge-agent.log
```

## Neutronサービスの動作を確認

controllerノードで`openstack network agent list`コマンドを実行してNeutronエージェントが正しく認識されており、稼働していることを確認します。

```
controller# source admin-openrc
controller# openstack network agent list -c Host -c Binary -c State
+------------+-------+---------------------------+
| Host       | State | Binary                    |
+------------+-------+---------------------------+
| controller | UP    | neutron-linuxbridge-agent |
| controller | UP    | neutron-metadata-agent    |
| controller | UP    | neutron-dhcp-agent        |
| controller | UP    | neutron-l3-agent          |
| compute    | UP    | neutron-linuxbridge-agent |
+------------+-------+---------------------------+
```

 ※コントローラーとコンピュートで追加され、neutron-linuxbridge-agentが正常に稼働していることが確認できれば問題ありません。念のためログも確認してください。

\clearpage

# Dashboardのインストールと確認（コントローラーノード）

クライアントマシンからブラウザーでOpenStack環境を操作可能なWebインターフェイスをインストールします。

## パッケージのインストール

controllerノードにDashboardをインストールします。

```
controller# apt install openstack-dashboard
```

## Dashboardの設定を変更

インストールしたDashboardの設定を行います。
すでにいくつかの設定は行われているので同じように設定がされているか確認し、されていない場合は設定を追加してください。言及していない設定はそのままで構いません。

```
controller# vi /etc/openstack-dashboard/local_settings.py
...
WEBROOT = '/horizon/'
LOGIN_URL = WEBROOT + 'auth/login/'
LOGOUT_URL = WEBROOT + 'auth/logout/'
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "compute": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = False
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
...
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'controller:11211',
    },
}
...
OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
...
TIME_ZONE = "Asia/Tokyo"  ← 変更(日本の場合)
```

\clearpage

次のコマンドで正しく設定を行ったか確認します。

```
controller# less /etc/openstack-dashboard/local_settings.py  | egrep -v "^\s*$|^\s*#"
```

変更した内容を反映させるため、Apacheとセッションストレージサービスを再起動します。

```
controller# service apache2 restart
```

## Dashboardにアクセス

コントローラーノードとネットワーク的に接続されているマシンからブラウザで以下URLに接続してOpenStackのログイン画面が表示されるか確認します。

※ブラウザで接続するマシンは予めDNSもしくは/etc/hostsにコントローラーノードのIPを記述しておく等コンピュートノードの名前解決を行っておく必要があります。ドメイン名でアクセスしないとVNCのコンソールが出ませんので注意してください。

```
http://controller/horizon/
```

※上記URLにアクセスしてログイン画面が表示され、ユーザーadminとdemoでログイン（パスワード:password）でログインできれば問題ありません。

\clearpage

## Externalネットワークの作成

OpenStack Dashboardにadminユーザーでログインして、Externalネットワークを作成します。
次のようにOpenStack Dashboardを操作して、Externalネットワークを作成してください。

1. OpenStack Dashboardにadminユーザーでログイン
2. 「管理 > ネットワーク > ネットワーク」でExternalネットワークを作成

ネットワーク

| 項目 | 設定 |
|-----|----- |
| 名前 | external-net |
| プロジェクト | admin |
| プロバイダネットワーク種別 | フラット |
| 物理ネットワーク | provider |
| 管理状態有効 | チェックを入れる |
| 共有 | チェックを入れない |
| 外部ネットワーク | チェックを入れる |
| サブネットの作成 | チェックを入れる |

3. 「次へ」ボタンを押下

サブネット

external-netで定義するIPアドレス及びその範囲は、OpenStackが実際に接続する外部接続可能なネットワークセグメントを指定します。

| 項目 | 設定 |
|-----|----- |
| サブネット名 | external-subnet |
| ネットワークアドレス | 10.0.0.0/24 |
| IPバージョン | IPv4  |
| ゲートウェイ | 10.0.0.1 |

4. 「次へ」ボタンを押下

サブネットの詳細

| 項目 | 設定 |
|-----|----- |
| DHCP有効 | チェックを入れない |
| IPアドレス割当プール | 10.0.0.177,10.0.0.190 |

IPアドレス割当プールはネットワークアドレスで定義したネットワーク範囲全てを割り当てても良い場合は定義する必要はありません。

4. 「作成」ボタンを押下

\clearpage

## インスタンスネットワークの作成

OpenStack Dashboardにdemoユーザーでログインして、インスタンスネットワークを作成します。

インスタンスネットワークはユーザーが作成したインスタンスを接続するネットワークです。外部からのアクセスは不可能ですが、同じインスタンスネットワーク上のインスタンスなどと通信が可能です。

インスタンスネットワークは管理者が共有ネットワークとして定義してユーザーに利用させることもできますが、多くの場合は各ユーザー毎にロールで定義した範囲でネットワークを自由に作成させることができます。

インスタンスネットワークとルーターを作成して、adminユーザーで作成したExternalネットワークと接続することにより、外から中への通信、つまり外部からインスタンスへのリモートアクセスなどが可能になります。

次のようにOpenStack Dashboardを操作して、インスタンスネットワークを作成してください。

1. OpenStack Dashboardにdemoユーザーでログイン
2. 「プロジェクト > ネットワーク > ネットワーク」でユーザーネットワークの作成

ネットワーク

| 項目 | 設定 |
|-----|----- |
| ネットワーク名 | user-net |
| 管理状態有効 | チェックを入れる |
| サブネットの作成 | チェックを入れる |

3. 「次へ」ボタンを押下

サブネット

| 項目 | 設定 |
|-----|----- |
| サブネット名 | user-subnet |
| ネットワークアドレス | 192.168.0.0/24 |
| IPバージョン | IPv4  |
| ゲートウェイIP | 192.168.0.1 |
| ゲートウェイなし | チェックを入れない |

\clearpage

4. 「次へ」ボタンを押下

サブネットの詳細

| 項目 | 設定 |
|-----|----- |
| DHCP有効 | チェックを入れる |
| IPアドレス割当プール | 未定義 |
| DNSサーバー | 8.8.8.8 |

DNSサーバーを複数指定したい場合は1行毎に記述します。IPアドレス割当プールはネットワークアドレスで定義したネットワーク範囲全てを割り当てても良い場合は定義する必要はありません。

5. 「作成」ボタンを押下

6. 「プロジェクト > ネットワーク > ルーター」でルーターを作成

| 項目 | 設定 |
|-----|----- |
| ルーター名 | myrouter |
| 管理状態有効 | チェックを入れる |
| 外部ネットワーク | external-net |

7. 「プロジェクト > ネットワーク > ルーター」で作成した「myrouter」をクリックして、インターフェイスを追加

| 項目 | 設定 |
|-----|----- |
| サブネット | user-net |
| IPアドレス | 未定義 |

\clearpage

## フレーバーの設定

フレーバーはインスタンスに設定する性能を定義するものです。従来のバージョンでは自動生成されていましたが、Pikeではデフォルトでフレーバーは定義されていません。

1. OpenStack Dashboardにadminユーザーでログイン
2. 「管理 > コンピュート > フレーバー」を選択
3. 「フレーバーの作成」ボタンを押下
4. フレーバー名、仮想CPU数、メモリー、ストレージサイズを定義
5. 「フレーバーの作成」ボタンを押下

CirrOSを動かすだけであれば、1vCPU,64MBメモリー,1GBストレージあれば充分です。
Ubuntuを動かす場合は、1vCPU,1GBメモリー,4GBストレージ以上のスペックが必要です。

## セキュリティグループの設定

OpenStackの上で動かすインスタンスのファイアウォール設定は、セキュリティグループで行います。ログイン後、次の手順でセキュリティグループを設定できます。

1. OpenStack Dashboardにdemoユーザーでログイン
2. 「プロジェクト > コンピュート > ネットワーク > セキュリティーグループ」を選択
3. 「ルールの管理」ボタンを押下
4. 「ルールの追加」で許可するルールを定義
5. 「追加」ボタンを押下

インスタンスに対してPingを実行したい場合はルールとしてすべてのICMPを、インスタンスにSSH接続したい場合はSSHをルールとしてセキュリティグループに追加してください。

セキュリティーグループは複数作成できます。作成したセキュリティーグループをインスタンスを起動する際に選択することで、セキュリティグループで定義したポートを解放したり、拒否したり、接続できるクライアントを制限することができます。

\clearpage

## キーペアの作成

OpenStackではインスタンスへのアクセスはデフォルトで公開鍵認証方式で行います。次の手順でキーペアを作成できます。

1. OpenStack Dashboardにdemoユーザーでログイン
2. 「プロジェクト > コンピュート > キーペア」をクリック
3. 「キーペアの作成」ボタンを押下
4. キーペア名を入力
5. 「キーペアの作成」ボタンを押下
6. キーペア（拡張子:pem）ファイルをダウンロード

既存のキーペアを使ってインスタンスにアクセスしたい場合は、キーペアのインポートを利用します。
catコマンドで所有する*.pubキーを標準出力して、その内容を貼り付けます。


## インスタンスの起動

前の手順でGlanceにCirrOSイメージを登録していますので、早速構築したOpenStack環境上でインスタンスを起動してみましょう。インスタンスの起動画面は、米印のついた項目が必須の設定です。

1. OpenStack Dashboardにdemoユーザーでログイン
2. 「プロジェクト > コンピュート > イメージ」をクリック
3. イメージ一覧から起動するOSイメージを選び、「インスタンスの起動」ボタンを押下
4. インスタンス名、フレーバー、ネットワーク、セキュリティーグループ、キーペアなどを設定
5. 最後に右下の「起動」ボタンを押下

初回起動時は少々時間がかかることがあります。

\clearpage

## Floating IPの設定

起動したインスタンスにFloating IPアドレスを設定することで、Dashboardのコンソール以外からインスタンスにアクセスできるようになります。インスタンスにFloating IPを割り当てるには次の手順で行います。

1. OpenStack Dashboardにdemoユーザーでログイン
2. 「プロジェクト > コンピュート > インスタンス」をクリック
3. インスタンスの一覧から割り当てるインスタンスをクリック
4. アクションメニューから「Floating IPの割り当て」をクリック
5. 「Floating IP割り当ての管理」画面のIPアドレスで「+」ボタンをクリック
6. 右下の「IPの確保」ボタンをクリック
7. 割り当てるIPアドレスとインスタンスを選択して右下の「割り当て」ボタンをクリック


## インスタンスへのアクセス

Floating IPを割り当てて、かつセキュリティグループの設定を適切に行っていれば、リモートアクセスできるようになります。セキュリティーグループでSSHを許可した場合、端末からSSH接続が可能になります（下記は実行例）。

```
client$ ping -c4 instance-floating-ip
client$ ssh -i mykey.pem cloud-user@instance-floating-ip
```

その他、適切なポートを開放してインスタンスへのPingを許可したり、インスタンスでWebサーバーを起動して外部PCからアクセスしてみましょう。

\clearpage


# 補足資料

## RHEL/CentOSとの違いについて

現在多くのOSでOpenStackのインストールをサポートしています。OpenStack公式にもディストリビューション別のセットアップ手順が公開されています。

<https://docs.openstack.org/pike/install/>

本書はUbuntuでのセットアップ手順を掲載していますが、細かい部分は異なるものの、OpenStackの設定などは共通な部分が多いためUbuntu以外のディストリビューションを使う場合も役立つと思います。本書が取り扱っている内容の中でUbuntuとRHEL/CentOSでの相違点についてここで取り上げたいと思います。

### サービスの取り扱いの違い

対象としているUbuntu 16.04およびRHEL 7/CentOS 7はいずれもsystemdが使われています。
しかし、これまでの慣習から、DebianベースのUbuntuは「インストールしたサービスは起動するように設定しても問題ないだろう」という思想から、パッケージのインストール直後はサービスが起動します。また、再起動後も自動的にサービスが起動するように設定されていることがほとんどです。

一方、RHELやCentOSでは`systemctl start`コマンドを使わない限り、サービスは起動しません。また、`systemctl enable`コマンドを使わない限り、サービスの永続化(自動起動)は設定されません。

|                    | Ubuntu |   RHEL   |
| ------------------ | ------ | -------- |
| サービスの初回起動 | される | されない |
| 自動起動設定       | される | されない |


### パッケージリポジトリー

自由に利用できるリポジトリーはそれぞれ次の通りです。

#### Ubuntu

[Ubuntu Cloud Archive](https://wiki.ubuntu.com/OpenStack/CloudArchive)に掲載されています。このリポジトリーはLTSバージョンのUbuntuで利用できます。

#### RHEL 7

[RDOプロジェクト](https://www.rdoproject.org/)が提供するリポジトリーのパッケージが利用できます。次のように実行すればPikeリリースの最新のパッケージが利用可能です。

```
# yum install -y yum install https://repos.fedorapeople.org/repos/openstack/openstack-pike/rdo-release-pike.rpm
```

RHEL7で構築する場合、パッケージのインストール前に、通常は有効化されていない次のリポジトリーを有効にする必要があります。

* rhel-7-server-optional-rpms
* rhel-7-server-rh-common-rpms
* rhel-7-server-extras-rpms

#### CentOS 7

[RDOプロジェクト](https://www.rdoproject.org/)が提供するリポジトリーのパッケージが利用できます。次のように実行すればPikeリリースの最新のパッケージが利用可能です。

```
# yum install https://repos.fedorapeople.org/repos/openstack/openstack-pike/rdo-release-pike.rpm
```

[CentOS Cloud SIG](https://wiki.centos.org/SpecialInterestGroup/Cloud)が提供しているパッケージも利用可能です。こちらのPikeリリースの最新版パッケージを利用する場合は、次のように実行します。

```
# yum install centos-release-openstack-pike
```

### OpenStack以外のパッケージやファイル名の違い

サービス名やモジュール名が異なります。設定ファイルもそれに合わせて異なります。

|          Ubuntu           |            RHEL            |
| ------------------------- | -------------------------- |
| apache2                   | httpd                      |
| libapache2-mod-wsgi       | mod_wsgi                   |
| /etc/apache2/apache2.conf | /etc/httpd/conf/httpd.conf |

### パッケージ名の違い

RHELやCentOSではパッケージ名の頭に`openstack-`がつきます。

|  Ubuntu  |        RHEL        |
| -------- | ------------------ |
| keystone | openstack-keystone |
| glance   | openstack-glance   |

### Neutronコンポーンメントにおける相違点

Neutron周りはパッケージの構成や追加でインストールするコンポーネントが異なります。

* controller-ubuntu

```
# apt install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent
```

* controller-rhel

```
# yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

* compute-ubuntu

```
# apt install neutron-linuxbridge-agent
```

* compute-rhel

```
# yum install openstack-neutron-linuxbridge ebtables ipset
```
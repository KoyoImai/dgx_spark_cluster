# DGX Spark Cluster構築

## 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QSFPケーブル2本 \
・LANケーブル6本 \
・[USB-Cハブ](https://www.ankerjapan.com/products/a8352?srsltid=AfmBOoq95ZKB998T5GecohoCODQpk4HWPhwSNI8mhbB-wakpWkvt89U1)（管理者nodeのRJ45増設用）

## 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \
・管理者nodeのみ研究室インターネットに接続しユーザーがログイン可能

## ステップ1 : IPアドレスの固定
### 管理者nodeでipアドレスを固定
まず、DGX Sparkは内蔵RJ45が1つしかないため、USB-Cハブを使って増設します。 \
現在の接続名を確認します。確認には以下のコマンドを使用してください。
```
mprg@spark-3894:~/Desktop$ nmcli con show
NAME                UUID                                  TYPE      DEVICE  
Wired connection 3  fc66a9f2-9f93-388b-b8a8-3e36f21c6947  ethernet  enP7s7  
lo                  7806b7b8-552c-4263-80df-aa8f0ae0be39  loopback  lo      
docker0             8619393a-f2dd-49b5-950d-f2febe719087  bridge    docker0 
Hotspot             6071fe71-0b54-4d73-9c62-7c4617411844  wifi      --      
MPRG                299f7e8b-3669-4de6-b871-ef5e8334e371  wifi      --      
MPRG 5GHz           d1ec31a2-3529-44a5-b938-3a05633cb2a5  wifi      --      
Wired connection 1  604acd0a-e284-35c1-9e37-9179391e3687  ethernet  --      
Wired connection 2  15d31a71-f3fb-3ce0-9132-3509b34bab24  ethernet  --      
Wired connection 4  e77d481f-837c-349b-8c5a-de01b5f2860e  ethernet  --      
Wired connection 5  8a5c2867-8304-3d08-8a1f-0ae0530676f1  ethernet  --      
Wired connection 6  6a80e8cb-9493-3ffb-b5fe-e318daead83b  ethernet  --      
mprg@spark-3894:~/Desktop$ 
```
接続名を確認すると、`ethernet  enP7s7`の名前が`Wired connection 3`とあり、これは研究室のネットワークに接続されています。 

`Wired connection 6`が増設したハブですので、こちらのIPアドレスを固定します。
`nmcli`コマンドでipアドレスを固定します。このとき、ipアドレスは`10.0.0.xxx`とし、`xxx`は重複をなくすため、研究室内で割り振られたDGX Sparkの番号にしています。
```
# 有線接続のIPアドレスを10.0.0.8に変更（接続名は上で確認した"Wired connection 6"）
mprg@spark-3894:~/Desktop$ sudo nmcli con mod "Wired connection 6" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.8/24 \
  ipv4.gateway "" \
  ipv4.dns ""


# 設定を反映
mprg@spark-3894:~/Desktop$ sudo nmcli con up "Wired connection 6"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/7)

# 確認
mprg@spark-3894:~/Desktop$ ip a show enx6c6e0705ec11
11: enx6c6e0705ec11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 6c:6e:07:05:ec:11 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.8/24 brd 10.0.0.255 scope global noprefixroute enx6c6e0705ec11
       valid_lft forever preferred_lft forever
    inet6 fe80::1dfa:8dc8:c3c9:818b/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```


### 計算用nodeでipアドレスを固定
管理者nodeと同様に計算用nodeもIPアドレスを固定する。
計算用nodeは4つあるため、全て以下の手順でIPアドレスを固定する。

まず、利用可能なネットワークとインターフェースのIPを確認します。
```
# 利用可能なネットワークの確認
mprg@spark-fb97:~/Desktop$ nmcli con show
NAME        UUID                                  TYPE      DEVICE  
MPRG        ee77595c-01d0-42a5-824f-c960106118da  wifi      wlP9s9  
有線接続 3  8905f334-ef05-360f-ac2e-a697a27a9901  ethernet  enP7s7  
lo          5f898638-7da0-4bf1-96f3-f5a75f1b5d56  loopback  lo      
docker0     90994e3c-4b64-4b86-b356-8caf531f0c77  bridge    docker0 
iPhone SE2  9000673e-cd0e-4f09-a8a4-3acd00ee5202  wifi      --      
有線接続 1  f2f71b9d-1533-34d0-90c2-38b166f58201  ethernet  --      
有線接続 2  6cae8701-bb5b-3695-a351-2c5ae41fe6e3  ethernet  --      
有線接続 4  427f80ca-feaa-34e4-add6-168638368b5c  ethernet  --      
有線接続 5  9cbf895e-c8dd-33c6-ac3f-7ceecbafa261  ethernet  --      

# インターフェースのIPを確認
mprg@spark-fb97:~/Desktop$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:97 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet6 fe80::1646:cdab:55d3:2585/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: enp1s0f0np0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 4c:bb:47:2f:fb:98 brd ff:ff:ff:ff:ff:ff
4: enp1s0f1np1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 4c:bb:47:2f:fb:99 brd ff:ff:ff:ff:ff:ff
5: enP2p1s0f0np0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 4c:bb:47:2f:fb:9c brd ff:ff:ff:ff:ff:ff
6: enP2p1s0f1np1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 4c:bb:47:2f:fb:9d brd ff:ff:ff:ff:ff:ff
7: wlP9s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether f8:3d:c6:56:9a:8c brd ff:ff:ff:ff:ff:ff
    altname wlP9p1s0
    inet 192.168.111.87/24 brd 192.168.111.255 scope global dynamic noprefixroute wlP9s9
       valid_lft 65609sec preferred_lft 65609sec
    inet6 fe80::7438:1e9e:91e8:25b0/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 96:6d:d0:21:f3:17 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

上記の結果から、`有線接続 3`がenP7s7（RJ45）に対応していることがわかるので、`有線接続 3`を指定してIPアドレスを固定します。
もし`有線接続1~5`のどれがenP7s7（RJ45）に対応しているかわからない場合、`nmcli con show "有線接続 x" | grep interface`を実行して確認してください。
（おそらく、`有線接続 3`がenP7s7（RJ45）に対応していると思います。）

```
# 有線接続のIPアドレスを10.0.0.15に変更（接続名は上で確認した"有線接続 3"）
mprg@spark-fb97:~/Desktop$ sudo nmcli con mod "有線接続 3" \
  connection.interface-name enP7s7 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.15/24 \
  ipv4.gateway "" \
  ipv4.dns ""

# 設定を有効化
mprg@spark-fb97:~/Desktop$ sudo nmcli con up "有線接続 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/181)

# 確認
mprg@spark-fb97:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:97 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.15/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
    inet6 fe80::1646:cdab:55d3:2585/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
mprg@spark-fb97:~/Desktop$ 
```
残りの計算用nodeについては、上記と同じ手順によってIPアドレスを固定してください（ここでは省略します）。
IPアドレスは`10.0.0.xxx`とし、`xxx`は研究室内のDGX Sparkに割り当てた番号とする。

### 各nodeで`/etc/hosts`を設定
管理者nodeと計算用nodeで`/etc/hosts`に以下の内容を追加します。
```
sudo bash -c 'cat >> /etc/hosts << EOF

# DGX Spark cluster
10.0.0.8    node8
10.0.0.15   node15
10.0.0.16   node16
10.0.0.17   node17
10.0.0.18   node18
10.0.0.8    spark-3894
EOF'
```
`/etc/hosts`の内容を追加したら、pingが通るかを確認してください。
```
ping -c 3 node8
ping -c 3 node15
ping -c 3 node16
ping -c 3 node17
ping -c 3 node18
```

## ステップ1.5 : 時間同期とその他
**[参考1:パッケージの自動更新(1)](https://qiita.com/ymbk990/items/cabfc383e1c5e35eb4f9)** \
**[参考2:パッケージの自動更新(2)](https://gihyo.jp/admin/serial/01/ubuntu-recipe/0671)** \
**[参考3:NTPサーバーによる時刻同期(1)](https://www.server-world.info/query?os=Ubuntu_24.04&p=ntp&f=1)**

job管理のため、管理者nodeと計算用nodeの時刻設定を揃えて多く必要がある。
なので、時刻設定の確認と調整を先にしましょう。

また、基本パッケージのインストールなどを行います。
まず、時刻同期を行います。
管理者nodeの設定から始めます。
まず、`pool ntp.nict.jp iburst`と`restrict 10.0.0.0 mask 255.255.255.0 nomodify notrap`を`/etc/ntpsec/ntp.conf`に追加します。
`/etc/ntpsec/ntp.conf`の内容は以下の通りです。
```
mprg@spark-3894:~$ sudo apt install -y ntpsec
mprg@spark-3894:~/Desktop$ cat /etc/ntpsec/ntp.conf 
# /etc/ntpsec/ntp.conf, configuration for ntpd; see ntp.conf(5) for help

driftfile /var/lib/ntpsec/ntp.drift
leapfile /usr/share/zoneinfo/leap-seconds.list

# To enable Network Time Security support as a server, obtain a certificate
# (e.g. with Let's Encrypt), configure the paths below, and uncomment:
# nts cert CERT_FILE
# nts key KEY_FILE
# nts enable

# You must create /var/log/ntpsec (owned by ntpsec:ntpsec) to enable logging.
#statsdir /var/log/ntpsec/
#statistics loopstats peerstats clockstats
#filegen loopstats file loopstats type day enable
#filegen peerstats file peerstats type day enable
#filegen clockstats file clockstats type day enable

# This should be maxclock 7, but the pool entries count towards maxclock.
tos maxclock 11

# Comment this out if you have a refclock and want it to be able to discipline
# the clock by itself (e.g. if the system is not connected to the network).
tos minclock 4 minsane 3

# Specify one or more NTP servers.

# Public NTP servers supporting Network Time Security:
# server time.cloudflare.com nts

# Use servers from the NTP Pool Project. Approved by Ubuntu Technical Board
# on 2011-02-08 (LP: #104525). See https://www.pool.ntp.org/join.html for
# more information.
#pool 0.ubuntu.pool.ntp.org iburst
#pool 1.ubuntu.pool.ntp.org iburst
#pool 2.ubuntu.pool.ntp.org iburst
#pool 3.ubuntu.pool.ntp.org iburst

# Use Ubuntu's ntp server as a fallback.
#server ntp.ubuntu.com
pool ntp.nict.jp iburst

# Access control configuration; see /usr/share/doc/ntpsec-doc/html/accopt.html
# for details.
#
# Note that "restrict" applies to both servers and clients, so a configuration
# that might be intended to block requests from certain clients could also end
# up blocking replies from your own upstream servers.

# By default, exchange time with everybody, but don't allow configuration.
restrict default kod nomodify nopeer noquery limited

# Local users may interrogate the ntp server more closely.
restrict 127.0.0.1
restrict ::1

# クラスタ内ノードからのアクセスを許可
restrict 10.0.0.0 mask 255.255.255.0 nomodify notrap
```
`/etc/ntpsec/ntp.conf`の内容を修正したら、NTPサーバーを再起動して確認を行ってください。
```
mprg@spark-3894:~/Desktop$ sudo systemctl restart ntpsec
mprg@spark-3894:~/Desktop$ sudo systemctl enable ntpsec
Synchronizing state of ntpsec.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable ntpsec
```
計算用nodeでNTPクライアントを設定するために、まず、`systemd-timesyncd`をインストールします。
```
sudo apt install -y systemd-timesyncd
```
続いて、`/etc/systemd/timesyncd.conf`に`NTP=10.0.0.8`を追記して、管理者nodeと時刻を同期するようにしてください。
最初は`#NTP=`とコメントアウトされていると思うので、コメントアウトを外して追記してください。
その上で、以下のコマンドで時刻動機を開始し、同期ができているかを確認してください。
```
mprg@spark-fb97:~$ sudo systemctl restart systemd-timesyncd
mprg@spark-fb97:~$ sudo systemctl enable systemd-timesyncd
mprg@spark-fb97:~$ timedatectl timesync-status
       Server: 10.0.0.8 (10.0.0.8)
Poll interval: 1min 4s (min: 32s; max 34min 8s)
         Leap: normal
      Version: 4
      Stratum: 2
    Reference: 85F3EEA3
    Precision: 1us (-24)
Root distance: 11.825ms (max: 5s)
       Offset: +8.073ms
        Delay: 312us
       Jitter: 0
 Packet count: 1
    Frequency: +172.328ppm
```
以上で時刻同期は完了です。
time zoneを同じにすることと自動更新の停止を全てのnodeで行ってください。
コマンドは以下の通りです。
```
# time zoneを全nodeで同じにする
sudo timedatectl set-timezone Asia/Tokyo

# 基本的なパッケージのインストール（基本的にsparkには入っていると思いますが一応）
sudo apt install -y ssh net-tools vim htop iotop tmux screen wget curl \
  build-essential cmake python3 python3-pip

# 自動更新の停止
mprg@spark-3894:~/Desktop$ sudo vim /etc/apt/apt.conf.d/10periodic
mprg@spark-3894:~/Desktop$ cat /etc/apt/apt.conf.d/10periodic
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Download-Upgradeable-Packages "0";
APT::Periodic::AutocleanInterval "0";
```

## ステップ2 : NFSサーバーの設定
**[参考1:NFSサーバーの設定(１)](https://tech.mntsq.co.jp/entry/2021/04/08/141423)** 

複数のnodeから同じファイル群を、同じパスで共有して使えるようにするため、NSFサーバーを導入します（MPRGクラスタｍｐNFSサーバーを使用しているはずです）。

### 管理者nodeでNFSを設定
まず、管理者nodeでNFSサーバーを設定します。
以下のコマンドを実行して、NFSサーバーをインストールしてください。
```
sudo apt install -y nfs-kernel-server
```
NFSサーバーをインストール後、`/etc/exports`に下記の内容を追加してください。
```
# 内容の追記
mprg@spark-3894:~/Desktop$ sudo bash -c 'cat >> /etc/exports << EOF

### NFS Mount of Home Directory
/home 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF'

# 内容の確認
mprg@spark-3894:~/Desktop$ cat /etc/exports 
# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

### NFS Mount of Home Directory
/home 10.0.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```
これで、`/home`を計算用nodeと共有することになります。
`/etc/exports`の変更を反映させるため、NFSサーバーを再起動します。
```
# NFSサーバーの再起動
mprg@spark-3894:~/Desktop$ sudo systemctl restart nfs-server
mprg@spark-3894:~/Desktop$ sudo systemctl enable nfs-server

# 結果を確認
mprg@spark-3894:~/Desktop$ sudo exportfs -v
/home         	10.0.0.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
mprg@spark-3894:~/Desktop$ 
```
（ここから下はあとから追加しました。`/home`ではなく`/home4cluster`で作業するための処理です。）
今後、`/home4cluster/usr1`や`/home4cluster/usr2`のようにユーザーごとのディレクトリを用意して運用する予定です。
以下のコマンドを実行して`/home4cluster`をマウントしてください。
```
mprg@spark-3894:~/Desktop$ sudo bash -c 'cat >> /etc/fstab << EOF

### Bind mount for cluster home
/home  /home4cluster  none  bind  0  0
EOF'
mprg@spark-3894:~/Desktop$ sudo mount -a
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mprg@spark-3894:~/Desktop$ sudo mount -a
mprg@spark-3894:~/Desktop$ sudo systemctl daemon-reload
```

### 計算用nodeでNFSクライアントを設定
計算用nodeでNFSクライアントを設定します。
以下のコマンドを実行して、NFSクライアントをインストールしてください。
```
sudo apt install -y nfs-common
```
NFSクライアントをインストールしたら、`/etc/fstab`にマウント設定を追記します。
以下の内容を`/etc/fstab`に追記してください。
```
# 内容を追記
mprg@spark-fb97:~$ sudo bash -c 'cat >> /etc/fstab << EOF

### NFS Mount of Home Directory
10.0.0.8:/home  /home4cluster  nfs  defaults,_netdev,vers=4.2,rsize=1048576,wsize=1048576  0  0
EOF'

# 内容の確認
mprg@spark-fb97:~$ cat /etc/fstab 
/dev/disk/by-uuid/d27bfd26-ff30-400e-9eca-9cdf73de9406 / ext4 errors=remount-ro 0 1
/dev/disk/by-uuid/9DA2-3597 /boot/efi vfat defaults 0 1
/swap.img none swap sw 0 0

### NFS Mount of Home Directory
10.0.0.8:/home  /home4cluster  nfs  defaults,_netdev,vers=4.2,rsize=1048576,wsize=1048576  0  0
```
最後に、実際にマウントを実行します。
```
sudo systemctl daemon-reload
sudo mount -a

# マウント確認
df -h | grep home4cluster
```


## ステップ3 : MUNGEの設定
**[参考1:MUNGEの設定(1)](https://qiita.com/kccs_takahiro-kawamura/items/bb0ffe731030aec3e4f5)**

Slurmの各node間で安全な認証と認可を提供するため、MUNGEを導入します。
MUNGEを導入し、全nodeで同じ`munge.key`を共有することで、Slurmが「このjob要求は、本当にそのユーザー、そのnodeから来たのか」を安全に確認することが可能になります。
以下では、MUNGEの設定手順をまとめます。

### 管理者nodeでMUNUGEを設定
まず、MUNGEをインストールします。
以下のコマンドを管理者nodeで実行してください。
```
sudo apt install -y munge libmunge-dev
```
MUNGEのインストールが完了したら、MUNGEの鍵ファイル`munge.key`を生成します。
`munge.key`を生成するため以下のコマンドを実行してください。
```
mprg@spark-3894:/home4cluster$ sudo dd if=/dev/random of=/etc/munge/munge.key bs=1024 count=1
1+0 records in
1+0 records out
1024 bytes (1.0 kB, 1.0 KiB) copied, 9.4736e-05 s, 10.8 MB/s
mprg@spark-3894:/home4cluster$ sudo chown munge:munge /etc/munge/munge.key
mprg@spark-3894:/home4cluster$ sudo chmod 400 /etc/munge/munge.key
mprg@spark-3894:/home4cluster$ sudo systemctl restart munge
mprg@spark-3894:/home4cluster$ sudo systemctl enable munge
Synchronizing state of munge.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable munge
mprg@spark-3894:/home4cluster$ 
```
`munge.key`の生成が完了したら、以下のコマンドで動作を確認してください。
```
mprg@spark-3894:/home4cluster$ munge -n | unmunge
STATUS:          Success (0)
ENCODE_HOST:     localhost (127.0.0.1)
ENCODE_TIME:     2026-04-01 20:17:56 +0900 (1775042276)
DECODE_TIME:     2026-04-01 20:17:56 +0900 (1775042276)
TTL:             300
CIPHER:          aes128 (4)
MAC:             sha256 (5)
ZIP:             none (0)
UID:             mprg (1000)
GID:             mprg (1000)
LENGTH:          0
```

### 計算nodeでMUNGEを設定
管理者nodeの`/etc/munge/munge.key`を全ての計算nodeにコピーします。
まず、管理者nodeの`munge.key`を全ての計算nodeにコピーするため、以下のコマンドを実行してください。
```
sudo scp /etc/munge/munge.key mprg@node15:/tmp/munge.key
sudo scp /etc/munge/munge.key mprg@node16:/tmp/munge.key
sudo scp /etc/munge/munge.key mprg@node17:/tmp/munge.key
sudo scp /etc/munge/munge.key mprg@node18:/tmp/munge.key
```
各計算nodeの`/tmp/`に`munge.key`をコピーしたら、以下のコマンドを全ての計算用nodeで実行してください。
```
# mungeのインストール
sudo apt install -y munge libmunge-dev

# /tmp/munge.key を /etc/munge/munge.key に移動
sudo mv /tmp/munge.key /etc/munge/munge.key

# 権限を変更
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

# mungeの再起動＆有効化
sudo systemctl restart munge
sudo systemctl enable munge
```

## ステップ4 : Slurmの構築
**[参考1:Slurmの構築(1)](https://qiita.com/kccs_takahiro-kawamura/items/bb0ffe731030aec3e4f5)** \
**[参考2:Slurmの構築(2)](https://slurm.schedmd.com/configurator.easy.html)** \
**[参考3:Slurmの構築(3)](https://slurm.schedmd.com/configurator.html)**

参考2と参考3で`slurm.conf`を作成できます。
複数の計算nodeを1つのクラスタとしてまとめて管理し、ユーザーのjob投入に対して適切な計算資源を自動で割り当てるためにSlurmを導入します。

### Slurmのビルドとインストール
まず、全てのnodeで必要なパッケージをインストールします。
以下のコマンドを実行してください。
```
sudo apt install build-essential cmake mailutils libmysqlclient-dev lua5.4 liblua5.4-dev libdbus-1-dev
```
上記のコマンドを実行すると、mail server configurationについて聞かれます。
「No configuration（設定なし）」を選択します。

続いて、Slurmソースをダウンロードするため、以下のコマンドを全てのnodeで順番に実行してください。
```
cd  ~
wget https://download.schedmd.com/slurm/slurm-24.11.3.tar.bz2
tar -jxvf slurm-24.11.3.tar.bz2
```
上記コマンド実行後、`slurm-24.11.3`ディレクトリができるので、以降はそこで作業します。
Slurmのビルドとインストールをするため、まず管理者nodeで以下のコマンドを実行してください。
```
cd ~/slurm-24.11.3/
export HAVEMYSQLCONFIG=/usr/bin/mysql_config
./configure --with-lua
make
sudo make install
```
続いて、計算用nodeで以下のコマンドを実行してください。
```
cd ~/slurm-24.11.3/
./configure
make
sudo make install
```

### slurm.confの生成
まず、管理者nodeで`/usr/local/etc/slurm.conf`を作成します。
`slurm.conf`は、`/usr/local/etc/slurm.conf/configurator.easy.html`を作成して生成可能です。
生成した`slurm.conf`は、一部の内容を修正します。
以下が、修正した`slurm.conf`の内容です。
```
mprg@spark-3894:~/slurm-24.11.3$ cat /usr/local/etc/slurm.conf
# slurm.conf file generated by configurator easy.html.
# Put this file on all nodes of your cluster.
# See the slurm.conf man page for more information.
#
ClusterName=dgx-spark-cluster
SlurmctldHost=spark-3894
#
#MailProg=/bin/mail
#MpiDefault=
#MpiParams=ports=#-#
ProctrackType=proctrack/cgroup
ReturnToService=1
SlurmctldPidFile=/var/run/slurmctld.pid
#SlurmctldPort=6817
SlurmdPidFile=/var/run/slurmd.pid
#SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurmd
SlurmUser=slurm
#SlurmdUser=root
StateSaveLocation=/var/spool/slurmctld
#SwitchType=
TaskPlugin=task/affinity,task/cgroup
#
#
# TIMERS
#KillWait=30
#MinJobAge=300
#SlurmctldTimeout=120
#SlurmdTimeout=300
#
#
# SCHEDULING
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core
#
#
GresTypes=gpu
# LOGGING AND ACCOUNTING
#AccountingStorageType=
#JobAcctGatherFrequency=30
#JobAcctGatherType=
#SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurm/slurmctld.log
#SlurmdDebug=info
SlurmdLogFile=/var/log/slurm/slurmd.log
#
#
# COMPUTE NODES
NodeName=node15 NodeHostname=spark-fb97 NodeAddr=10.0.0.15 CPUs=20 RealMemory=119543 Sockets=2 CoresPerSocket=10 ThreadsPerCore=1 Gres=gpu:blackwell:1 State=UNKNOWN
NodeName=node16 NodeHostname=spark-4440 NodeAddr=10.0.0.16 CPUs=20 RealMemory=119543 Sockets=2 CoresPerSocket=10 ThreadsPerCore=1 Gres=gpu:blackwell:1 State=UNKNOWN
NodeName=node17 NodeHostname=spark-755c NodeAddr=10.0.0.17 CPUs=20 RealMemory=119543 Sockets=2 CoresPerSocket=10 ThreadsPerCore=1 Gres=gpu:blackwell:1 State=UNKNOWN
NodeName=node18 NodeHostname=spark-07a2 NodeAddr=10.0.0.18 CPUs=20 RealMemory=119543 Sockets=2 CoresPerSocket=10 ThreadsPerCore=1 Gres=gpu:blackwell:1 State=UNKNOWN

PartitionName=pair1 Nodes=node15,node16 MaxNodes=2 Default=YES MaxTime=INFINITE State=UP
PartitionName=pair2 Nodes=node17,node18 MaxNodes=2 MaxTime=INFINITE State=UP
```
管理者nodeで作成した`slurm.conf`を全ての計算用nodeに配布してください。
```
sudo scp /usr/local/etc/slurm.conf mprg@node15:/tmp/slurm.conf
sudo scp /usr/local/etc/slurm.conf mprg@node16:/tmp/slurm.conf
sudo scp /usr/local/etc/slurm.conf mprg@node17:/tmp/slurm.conf
sudo scp /usr/local/etc/slurm.conf mprg@node18:/tmp/slurm.conf
```
管理者nodeから各計算用nodeに`slurm.conf`をコピーしたら、配置場所を修正してください。
以下のコマンドを計算用node全てで実行してください。
```
sudo mv /tmp/slurm.conf /usr/local/etc/slurm.conf
```

### Slurmユーザーの作成
slurmユーザーを作成します。
まず、管理者nodeで以下のコマンドを実行します。
パスワードは、`Ki**************`です。
```
sudo adduser slurm --uid 1099
sudo gpasswd -a slurm sudo
sudo usermod -aG syslog slurm
```
続いて、全ての計算用nodeで以下のコマンドを実行してください。
```
sudo useradd -u 1099 -d /nonexistent -s /usr/sbin/nologin -M slurm
sudo gpasswd -a slurm sudo
sudo usermod -aG syslog slurm
```

### gres.confの作成
**[参考1:gres.confの作成(1)](https://slurm.schedmd.com/gres.conf.html)**

計算nodeで`gres.conf`を作成します。
`gres.conf`を作成することで、どのnodeにどのGPUが載っているかを正確に認識することが可能になります。
全ての計算nodeで以下の内容の`gres.conf`を作成してください。
```
mprg@spark-fb97:~/slurm-24.11.3$ cat /usr/local/etc/gres.conf 
NodeName=node15 AutoDetect=off Name=gpu Type=blackwell File=/dev/nvidia0
NodeName=node16 AutoDetect=off Name=gpu Type=blackwell File=/dev/nvidia0
NodeName=node17 AutoDetect=off Name=gpu Type=blackwell File=/dev/nvidia0
NodeName=node18 AutoDetect=off Name=gpu Type=blackwell File=/dev/nvidia0

mprg@spark-fb97:~/slurm-24.11.3$ 
```

### 必要なディレクトリの作成と権限付与
全てのnodeで以下のコマンドを実行してディレクトリの作成などを行ってください。
これによって、Slurm Daemonが起動時に必要とする保存先・作業領域・ログ出力先をあらかじめ用意し、必要な権限を付与しておけます。
```
sudo mkdir -p /var/spool/slurmd
sudo mkdir -p /var/spool/slurmctld
sudo mkdir -p /var/log/slurm
sudo chown -R slurm:slurm /var/spool/slurmd
sudo chown -R slurm:slurm /var/spool/slurmctld
sudo chown -R slurm:slurm /var/log/slurm
sudo chown slurm:slurm /usr/local/etc/slurm.conf
sudo chmod 644 /usr/local/etc/slurm.conf
```

### Slurm Daemonの登録・起動
Slurm Daemonの登録と起動を行います。
Daemonとは、Kinux系OSにおいて、バックグラウンドに常駐し、ネットワークサービスやシステム管理などの特定タスクを自動で処理するプログラムです。
ですので、Slurm Daemonを登録・起動することで、Slurmの機能を常時バックグラウンドで動かしておけます。
Slurmには`slurmd`、`slurmctld`、`slurmbd`の3種類のDaemonがあります。
```
slurmctld : Slurmの中央管理を行うDaemonです。他のSlurm Daemonや資源を監視し、jobの受付や資源の割当を行う。
slurmd : 計算node上で動くDaemonです。
slurmdbd : データベースを保存するためのDaemonです。
```

まず、管理者nodeで`slurmctld`を登録・起動します。
以下のコマンドを実行してください。
```
cd ~/slurm-24.11.3/etc/
sudo cp slurmctld.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable slurmctld
sudo systemctl start slurmctld

# 状態確認
sudo systemctl status slurmctld
```
状態確認で`Active: active (running)`が出ていれば問題ないです。
（注意：最初の`slurm.conf`だとnode名が違ったため、修正しました。途中エラーが出まくって色々いじったので、もしかしたら、上記のコマンドでエラーが出るかもしれないです。）

続いて、計算用nodeでもSlurm Daemonの起動と登録を行います。
（注意：エラー解決で色々いじったので、記入漏れがあるかもしれないです。）
```
# cgroup.confの作成
sudo bash -c 'cat > /usr/local/etc/cgroup.conf << EOF
CgroupPlugin=autodetect
EOF'

cd ~/slurm-24.11.3/etc/
sudo cp slurmd.service /etc/systemd/system/
sudo systemctl daemon-reload

sudo systemctl enable slurmd
sudo systemctl start slurmd
sudo systemctl status slurmd
```
管理者nodeで以下を実行して結果を確認してください。
```
mprg@spark-3894:~/slurm-24.11.3$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
pair1*       up   infinite      2   idle node[15-16]
pair2        up   infinite      2   idle node[17-18]
mprg@spark-3894:~/slurm-24.11.3$

mprg@spark-3894:~/slurm-24.11.3$ srun --partition=pair1 --nodes=1 hostname
spark-fb97
mprg@spark-3894:~/slurm-24.11.3$ srun --partition=pair1 --nodes=2 hostname
spark-4440
spark-fb97
mprg@spark-3894:~/slurm-24.11.3$ srun --partition=pair1 --nodes=1 --gres=gpu:1 nvidia-smi
Fri Apr  3 14:11:57 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   35C    P8              4W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3338      G   /usr/lib/xorg/Xorg                      133MiB |
|    0   N/A  N/A            3491      G   /usr/bin/gnome-shell                     93MiB |
|    0   N/A  N/A            4415      G   .../7965/usr/lib/firefox/firefox        674MiB |
|    0   N/A  N/A            9715      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
mprg@spark-3894:~/slurm-24.11.3$ srun --partition=pair2 --nodes=1 hostname
spark-755c
mprg@spark-3894:~/slurm-24.11.3$ 

```

## ステップ5:SSH設定
利用者が計算用nodeに直接アクセスできないように、SSH設定を行います。
まず、管理者nodeで以下のコマンドを実行してください。
```
sudo systemctl enable ssh
sudo systemctl start ssh
```
続いて、計算用nodeで以下のコマンドを実行してください。
```
echo "AllowUsers mprg" | sudo tee -a /etc/ssh/sshd_config
tail -3 /etc/ssh/sshd_config
sudo systemctl restart ssh
```

## ステップ6:ユーザーの作成
クラスタに接続してjobを投げるユーザーの作成などを行います。
管理者nodeで以下のコマンドを実行し、ユーザーの登録をしてください。
```
# ホームディレクトリを /home4cluster/kouyou に指定してユーザー作成
sudo useradd -m -d /home4cluster/kouyou -s /bin/bash kouyou

# パスワードを設定
sudo passwd kouyou

# 確認
id kouyou
ls /home4cluster/
```

## ステップ7:Singularityの導入
**[参考1:Singularityの導入](https://note.com/holyday_mylife/n/n58cf55315d46)** \
**[参考2:Singularityの導入](https://gist.github.com/muripoLife/d21fe546d530f0e0474ad1b053b5b084)** \
**[参考3:Singularityの導入](https://docs.sylabs.io/guides/3.0/user-guide/installation.html)**

MPRGクラスターと同様にSingularityを導入します。
まず、管理者nodeで以下のコマンドを実行して、必要なパッケージをインストールしてください。
```
sudo apt-get update
sudo apt-get install -y autoconf automake cryptsetup fuse2fs git fuse \
  libfuse-dev libseccomp-dev libtool pkg-config runc squashfs-tools \
  squashfs-tools-ng uidmap wget zlib1g-dev libsubid-dev
```
続いて、`conmon`をインストールします。
以下のコマンドを実行してください。
```
sudo apt-get install -y conmon
```
次に、`Go`をインストールします。
以下のコマンドを実行します。
```
export VERSION=1.24.1 OS=linux ARCH=arm64

wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz \
  https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz

sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz

# パスを設定
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee -a /etc/profile

# 現在のセッションにも反映
export PATH=$PATH:/usr/local/go/bin

# 確認
go version
```
最後に、Singularityのソースをクローン・ビルドします。
以下のコマンドを実行してクローンしてください。
```
cd ~
git clone --recurse-submodules https://github.com/sylabs/singularity.git
cd singularity
git submodule update --init
git checkout --recurse-submodules v4.3.0
```
クローンが完了したら、ビルドします。
以下のコマンドを実行してください。
```
./mconfig
make -C builddir -j 30
sudo make -C builddir install
```
Singularityのバージョン確認等を行います。
以下のコマンドを実行してください。
```
singularity --version
sudo tee /etc/apparmor.d/singularity-ce << 'EOF'
# Permit unprivileged user namespace creation for SingularityCE starter
abi <abi/4.0>,
include <tunables/global>

profile singularity-ce /usr/local/libexec/singularity/bin/starter{,-suid} flags=(unconfined) {
  userns,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/singularity-ce>
}
EOF
sudo systemctl reload apparmor
```
ここまでで、管理者nodeでSingularityの導入が完了しました。

ここからは、全ての計算用nodeでSingularityの導入を行っていきます。
基本的な手順は管理者nodeと同じです。
以下のコマンドを実行して必要なパッケージをインストールしてください。
```
sudo apt-get update
sudo apt-get install -y autoconf automake cryptsetup fuse2fs git fuse \
  libfuse-dev libseccomp-dev libtool pkg-config runc squashfs-tools \
  squashfs-tools-ng uidmap wget zlib1g-dev libsubid-dev conmon
```
必要なパッケージをインストールしたら、Goをインストールします。
以下のコマンドを実行してください。
```
export VERSION=1.24.1 OS=linux ARCH=arm64
wget -O /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz \
  https://dl.google.com/go/go${VERSION}.${OS}-${ARCH}.tar.gz
sudo tar -C /usr/local -xzf /tmp/go${VERSION}.${OS}-${ARCH}.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee -a /etc/profile
export PATH=$PATH:/usr/local/go/bin
go version
```
Goのインストールが完了したら、Singularityのクローンとビルドをします。
以下のコマンドを実行してください。
```
cd ~
git clone --recurse-submodules https://github.com/sylabs/singularity.git
cd singularity
git submodule update --init
git checkout --recurse-submodules v4.3.0
./mconfig
make -C builddir -j 30
sudo make -C builddir install
```
最後にAppArmorプロファイルの設定を行います。
以下のコマンドを実行してください。
```
sudo tee /etc/apparmor.d/singularity-ce << 'EOF'
# Permit unprivileged user namespace creation for SingularityCE starter
abi <abi/4.0>,
include <tunables/global>

profile singularity-ce /usr/local/libexec/singularity/bin/starter{,-suid} flags=(unconfined) {
  userns,

  include if exists <local/singularity-ce>
}
EOF
sudo systemctl reload apparmor
```
以上で、管理者nodeと計算用nodeにSingularityのインストールが完了しました。
確認をする場合は以下で確認してください。
```
singularity --version
```

## ステップ8:DGX Sparkの2台接続
**[参考1:DGX Sparkの2台接続](https://build.nvidia.com/spark/connect-two-sparks/stacked-sparks)** \
**[参考２:DGX Sparkの2台接続](https://dev.classmethod.jp/articles/dgx-spark-two-node-clustering/)**

1jobを最大2nodeで動かせるように、DGX Spark2台をQSFPケーブルで接続し、動作可能な状態にします。
まず、QSFPケーブルで2台のDGX Sparkを接続してください。
ここでは、`node15`と`node16`を接続して作業を勧めていきます。

QSFPケーブルで2台のDGX Sparkを接続したら、ネットワークインターフェースの設定を行います。
NVIDIA公式に従って、自動IP割り当てで設定します。
まず、2台が接続できているかを、以下のコマンドで確認してください。
```
mprg@spark-fb97:~/singularity$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
```
`enp1s0f0np0 (Up)`が`Up`となっているため、このインターフェースを使用します。
以下のコマンドを、node15とnode16の両方で実行してください。
```
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      link-local: [ ipv4 ]
    enp1s0f1np1:
      link-local: [ ipv4 ]
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```
node15とnode16でIPアドレスを確認します。
```
# node15
mprg@spark-fb97:~/singularity$ ip addr show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:fb:98 brd ff:ff:ff:ff:ff:ff
    inet 169.254.20.98/16 brd 169.254.255.255 scope link noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2f:fb98/64 scope link 
       valid_lft forever preferred_lft forever

# node16
mprg@spark-4440:~/singularity$ ip addr show enp1s0f0np0
3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:44:41 brd ff:ff:ff:ff:ff:ff
    inet 169.254.201.68/16 brd 169.254.255.255 scope link noprefixroute enp1s0f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ebb:47ff:fe2f:4441/64 scope link 
       valid_lft forever preferred_lft forever
```
node15では`169.254.20.98`、node16では`169.254.201.68`が割り当てられているのが確認できます。
一応、pingが通るかを確認します。
```
# node15
mprg@spark-fb97:~/singularity$ ping -c 3 169.254.201.68
PING 169.254.201.68 (169.254.201.68) 56(84) bytes of data.
64 bytes from 169.254.201.68: icmp_seq=1 ttl=64 time=1.23 ms
64 bytes from 169.254.201.68: icmp_seq=2 ttl=64 time=0.908 ms
64 bytes from 169.254.201.68: icmp_seq=3 ttl=64 time=1.19 ms
--- 169.254.201.68 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.908/1.108/1.226/0.142 ms
mprg@spark-fb97:~/singularity$ 

# node16
mprg@spark-4440:~/singularity$ ping -c 3 169.254.20.98
PING 169.254.20.98 (169.254.20.98) 56(84) bytes of data.
64 bytes from 169.254.20.98: icmp_seq=1 ttl=64 time=0.849 ms
64 bytes from 169.254.20.98: icmp_seq=2 ttl=64 time=1.06 ms
64 bytes from 169.254.20.98: icmp_seq=3 ttl=64 time=0.724 ms

--- 169.254.20.98 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2040ms
rtt min/avg/max/mdev = 0.724/0.878/1.063/0.139 ms
mprg@spark-4440:~/singularity$ 
```
node15とnode16の両方においてpingが通っていることが確認できます。

ここまでで、IPアドレスの固定をしたが、再起動時にIPアドレスがリセットされる恐れがあるので、手動設定に切り替える。
全ての計算nodeで以下のコマンドを実行してください。
```
# node15
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.1/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply

ip addr show enp1s0f0np0
```
```
# node16
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.1.2/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply

ip addr show enp1s0f0np0
```
```
# node17
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.1/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply

ip addr show enp1s0f0np0
```
```
# node18
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 10.0.2.2/24
      dhcp4: no
EOF

sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply

ip addr show enp1s0f0np0
```
手動でIPアドレスを固定しました。
一旦、QSFPケーブルで通信速度が向上しているかを確認します。
node15とnode16で確認をするために、以下のコマンドを2つのnodeで実行してください。
```
sudo apt install -y iperf3
```
`iperf3`をインストールしたら、node16でのみ以下のコマンドを実行してください。
```
iperf3 -s
```
その後、node15で以下のコマンドを実行して速度を比較してください。
```
# Ethernet経由
iperf3 -c 10.0.0.16 -t 10 -P 8

# QSFP経由
iperf3 -c 10.0.1.2 -t 10 -P 8
```
速度比較の結果を見ると、並列処理するとQSFP経由のほうが速度向上が早いことがわかると思います。

ここからは、NCCLの設定を行っていきます。
全てのnodeで以下を実行してください。
```
sudo bash -c 'cat >> /etc/hosts << EOF

# QSFP high-speed network
10.0.1.1   node15-qsfp
10.0.1.2   node16-qsfp
10.0.2.1   node17-qsfp
10.0.2.2   node18-qsfp
EOF'
```
その後、node15で以下のコマンドを実行してください。
```
# SSH鍵を生成（既にある場合はスキップ）
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# node16に公開鍵をコピー
ssh-copy-id -i ~/.ssh/id_rsa.pub mprg@10.0.0.16

# 確認
ssh mprg@10.0.1.2 hostname
```
次に以下のコマンドをnode15とnode16の両方で実行してNCCLのビルドを行ってください。
```
# 依存パッケージのインストール
sudo apt-get update && sudo apt-get install -y libopenmpi-dev

# NCCLをクローン・ビルド
git clone -b v2.28.9-1 https://github.com/NVIDIA/nccl.git ~/nccl/
cd ~/nccl/
make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"

# 環境変数の設定
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build/"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64/:$MPI_HOME/lib:$LD_LIBRARY_PATH"
```
NCCLのビルドが完了したら、テストします。
以下のコマンドをnode15とnode16実行してください。
```
git clone https://github.com/NVIDIA/nccl-tests.git ~/nccl-tests/
cd ~/nccl-tests/
make MPI=1
```
以上は完了したら、実際にテスト行います。
node15で以下のコマンドを実行してください。
```
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build/"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64/:$MPI_HOME/lib:$LD_LIBRARY_PATH"

export UCX_NET_DEVICES=enp1s0f0np0
export NCCL_SOCKET_IFNAME=enp1s0f0np0
export OMPI_MCA_btl_tcp_if_include=enp1s0f0np0

# all_gatherテスト（node15のIPとnode16のIPを指定）
mpirun -np 2 -H 10.0.1.1:1,10.0.1.2:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf

# 16GBバッファでのテスト
mpirun -np 2 -H 10.0.1.1:1,10.0.1.2:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```
以上でnode15とnode16のテストは終了です。

ここから、同様の手順でnode17とnode18も設定します。
以下のコマンドを実行してください。
```
# node17
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub mprg@10.0.0.18
ssh mprg@10.0.0.18 hostname

# node17 & node18
sudo apt-get update && sudo apt-get install -y libopenmpi-dev
git clone -b v2.28.9-1 https://github.com/NVIDIA/nccl.git ~/nccl/
cd ~/nccl/
make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"

export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build/"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64/:$MPI_HOME/lib:$LD_LIBRARY_PATH"

# node17 & node18
git clone https://github.com/NVIDIA/nccl-tests.git ~/nccl-tests/
cd ~/nccl-tests/
make MPI=1

# node17
export UCX_NET_DEVICES=enp1s0f0np0
export NCCL_SOCKET_IFNAME=enp1s0f0np0
export OMPI_MCA_btl_tcp_if_include=enp1s0f0np0

mpirun -np 2 -H 10.0.2.1:1,10.0.2.2:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```
以上でQSFP接続とNCCL設定については終了です。



## ステップ9:DGX Sparkを4台使用可能に設定（w/o QSFP）
QSFP接続ではなく、Ethernet経由で4台同時に使用できるように設定を変更します。
そのために、`/usr/local/etc/slurm.conf`の設定を修正します。
管理者nodeで以下の内容を`/usr/local/etc/slurm.conf`に追加してください。
```
PartitionName=pair1 Nodes=node15,node16 MaxNodes=2 Default=YES MaxTime=INFINITE State=UP
PartitionName=pair2 Nodes=node17,node18 MaxNodes=2 MaxTime=INFINITE State=UP
PartitionName=all Nodes=node15,node16,node17,node18 MaxNodes=4 MaxTime=INFINITE State=UP
```
管理者nodeで`slurm.conf`の修正が完了したら、それを計算nodeに配布します。
以下のコマンドを管理者nodeで実行してください。
```
# 全計算ノードに配布（mgmtで実行）
for node in node15 node16 node17 node18; do
  sudo scp /usr/local/etc/slurm.conf mprg@${node}:/tmp/slurm.conf
done
```
各計算nodeに配布したら、計算node側で以下を実行してください。
```
sudo mv /tmp/slurm.conf /usr/local/etc/slurm.conf
```
管理者nodeで以下を実行して`slurmctld`を再起動してください。
```
sudo systemctl restart slurmctld
sinfo
```
全ての計算nodeで以下を実行してください。
```
sudo systemctl restart slurmd
```
管理者nodeで状態を確認してください。
```
mprg@spark-3894:~/singularity$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
pair1*       up   infinite      2   idle node[15-16]
pair2        up   infinite      2   idle node[17-18]
all          up   infinite      4   idle node[15-18]
mprg@spark-3894:~/singularity$ 
```
各partitionがidle状態になっているので成功していることがわかります。
4台全てを使用するjobを投入して確認します。
```
# 4台全てでhostnameを実行
mprg@spark-3894:~/singularity$ srun --partition=all --nodes=4 hostname
spark-fb97
spark-4440
spark-07a2
spark-755c
mprg@spark-3894:~/singularity$
```

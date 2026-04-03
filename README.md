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
```

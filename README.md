# DGX Spark Cluster構築

## 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QFSPケーブル2本 \
・LANケーブル5本 \
・USB-Cハブ（管理者nodeのRJ45増設用）

## 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \
・管理者nodeのみ研究室インターネットに接続しユーザーがログイン可能

## ステップ1
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
残りの計算用nodeについては、上記と同じ手順によってIPアドレスを固定する。
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


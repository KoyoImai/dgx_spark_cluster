# DGX Spark Cluster構築

# 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QFSPケーブル2本 \
・LANケーブル5本 \

# 構成（予定）
・管理者node : DGX Spark 08 \
・計算用node : DGX Spark 15 ~ 18 \
・管理者nodeと計算用nodeをRJ45 Ethernet スイッチ経由で接続 \

## ステップ1
### 管理者nodeでipアドレスを固定
まず、現在の接続名を確認します。確認には以下のコマンドを使用してください。
```
mprg@spark-3894:~/Desktop$ nmcli con show
NAME                UUID                                  TYPE      DEVICE  
MPRG                299f7e8b-3669-4de6-b871-ef5e8334e371  wifi      wlP9s9  
Wired connection 3  fc66a9f2-9f93-388b-b8a8-3e36f21c6947  ethernet  enP7s7  
lo                  7806b7b8-552c-4263-80df-aa8f0ae0be39  loopback  lo      
docker0             8619393a-f2dd-49b5-950d-f2febe719087  bridge    docker0 
Hotspot             6071fe71-0b54-4d73-9c62-7c4617411844  wifi      --      
MPRG 5GHz           d1ec31a2-3529-44a5-b938-3a05633cb2a5  wifi      --      
Wired connection 1  604acd0a-e284-35c1-9e37-9179391e3687  ethernet  --      
Wired connection 2  15d31a71-f3fb-3ce0-9132-3509b34bab24  ethernet  --      
Wired connection 4  e77d481f-837c-349b-8c5a-de01b5f2860e  ethernet  --      
Wired connection 5  8a5c2867-8304-3d08-8a1f-0ae0530676f1  ethernet  --      
mprg@spark-3894:~/Desktop$
```
接続名を確認すると、`ethernet  enP7s7`の名前が`Wired connection 3`とわかります。

`nmcli`コマンドでipアドレスを固定します。このとき、ipアドレスは`10.0.0.xxx`とし、`xxx`は重複をなくすため、研究室内で割り振られたDGX Sparkの番号にしています。

```
# 有線接続のIPアドレスを10.0.0.8に変更（接続名は上で確認した"Wired connection 3"）
mprg@spark-3894:~/Desktop$ sudo nmcli con mod "Wired connection 3" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.8/24 \
  ipv4.gateway "" \
  ipv4.dns ""


# 設定を反映
mprg@spark-3894:~/Desktop$ sudo nmcli con up "Wired connection 3"
接続が正常にアクティベートされました (D-Bus アクティブパス: /org/freedesktop/NetworkManager/ActiveConnection/7)


# 確認
mprg@spark-3894:~/Desktop$ ip a show enP7s7
2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:2f:38:94 brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 10.0.0.8/24 brd 10.0.0.255 scope global noprefixroute enP7s7
       valid_lft forever preferred_lft forever
```


### 計算用nodeでipアドレスを固定



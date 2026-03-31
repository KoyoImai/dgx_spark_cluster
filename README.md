# DGX Spark Cluster構築

# 前準備
・DGX Spark5台（管理者node用1台，計算用node4台） \
・RJ45 Ethernet スイッチ（8ポート） \
・QFSPケーブル2本 \
・LANケーブル5本 \

# 構成（予定）
・管理者node : DGX Spark 08
・計算用node : DGX Spark 15 ~ 18

## ステップ1
### 管理者nodeでipアドレスを固定
```
sudo nmcli con mod "Wired connection 3" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.8/24 \
  ipv4.gateway "" \
  ipv4.dns ""

```


### 計算用nodeでipアドレスを固定



# LLMの推論速度を比較5
QSFPケーブルのみを使用してDGX Sparkを3台接続し、スイッチなしで仮想的な大容量GPUを構築する。
llama.cppのRPC機能を利用し、node15をメインノード、node16・node17をRPCサーバーとするスター型構成で推論速度を検証・比較する。

## ネットワーク構成
node15をハブとするスター型QSFP接続を採用する。
各DGX SparkはQSFPポートを2つ持つ（`enp1s0f0np0`と`enp1s0f1np1`）。
接続の注意点として、**同じポート同士しか接続できない**（`enp1s0f0np0`↔`enp1s0f0np0`、`enp1s0f1np1`↔`enp1s0f1np1`）。

```
node16(enp1s0f0np0) ←QSFP 200Gbps→ node15(enp1s0f0np0)
node17(enp1s0f1np1) ←QSFP 200Gbps→ node15(enp1s0f1np1)
```

各インターフェースのIPアドレス：
| node | インターフェース | IPアドレス |
|---|---|---|
| node15 | enp1s0f0np0 | 10.0.1.1 |
| node15 | enp1s0f1np1 | 10.0.4.1 |
| node16 | enp1s0f0np0 | 10.0.1.2 |
| node17 | enp1s0f1np1 | 10.0.4.2 |

## 推論速度比較結果

### 測定条件1
- モデル：Qwen3-235B-A22B Q2_K（79.80 GiB）
- pp128：128トークンのプロンプト処理速度（t/s）
- tg256：256トークンの生成速度（t/s）
- 測定回数：3回の平均（`-r 3`）

### 結果一覧1
| パターン | node数 | NW | backend | pp128 (t/s) | tg256 (t/s) |
|---|---:|---|---|---:|---:|
| 1台 | 1 | - | CUDA | 108.30 ± 6.94 | 15.63 ± 0.33 |
| 2台（QSFP） | 2 | QSFP 200Gbps | CUDA,RPC | 108.85 ± 7.27 | 15.99 ± 0.25 |
| 3台（QSFP） | 3 | QSFP 200Gbps | CUDA,RPC | 140.73 ± 5.52 | 16.91 ± 0.02 |


## ステップ1：ネットワーク設定

### QSFPインターフェースのIPアドレス設定
node15では既存の`40-cx7.yaml`に`enp1s0f0np0`（10.0.1.1）の設定が存在する。
新たに`enp1s0f1np1`の設定を追加する場合は、既存ファイルを壊さないよう**別ファイル**（`41-cx7-p2.yaml`）として作成する。

node15で`enp1s0f1np1`のIPアドレスを設定する（既存の`40-cx7.yaml`はそのまま残す）。
```
# 既存設定の確認
cat /etc/netplan/40-cx7.yaml

# enp1s0f1np1の設定を別ファイルとして追加
sudo tee /etc/netplan/41-cx7-p2.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 10.0.4.1/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/41-cx7-p2.yaml
sudo netplan apply
```

node17で`enp1s0f1np1`のIPアドレスを設定する。
```
sudo tee /etc/netplan/41-cx7-p2.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f1np1:
      addresses:
        - 10.0.4.2/24
      dhcp4: no
EOF
sudo chmod 600 /etc/netplan/41-cx7-p2.yaml
sudo netplan apply
```

### 疎通確認
node15からnode16とnode17への疎通確認を行う。
```
# node15からnode16方向
ping -c 3 10.0.1.2

# node15からnode17方向
ping -c 3 10.0.4.2
```

## ステップ2：1台推論（動作確認）
まず1台で動作確認を行う。node15で以下を実行する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/llama-bench \
  --model /home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf \
  --n-gpu-layers 99 \
  -p 128 -n 256 -r 3
```

## ステップ3：2台推論（QSFP）
node16のみRPCサーバーを起動し、node15からllama-benchを実行する。

### RPCサーバーの起動
node16でRPCサーバーを起動する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/rpc-server --host 10.0.1.2 --port 50052
```

### llama-benchの実行
node15で以下を実行する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/llama-bench \
  --model /home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf \
  --n-gpu-layers 99 \
  --rpc 10.0.1.2:50052 \
  -p 128 -n 256 -r 3
```

## ステップ4：3台推論（QSFP スター型）
node16とnode17でRPCサーバーを起動し、node15からllama-benchを実行する。

### RPCサーバーの起動
node16でRPCサーバーを起動する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/rpc-server --host 10.0.1.2 --port 50052
```

node17でRPCサーバーを起動する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/rpc-server --host 10.0.4.2 --port 50052
```

### llama-benchの実行
RPCサーバーの起動を確認した後、node15でllama-benchを実行する。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/llama-bench \
  --model /home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf \
  --n-gpu-layers 99 \
  --rpc 10.0.1.2:50052,10.0.4.2:50052 \
  -p 128 -n 256 -r 3
```

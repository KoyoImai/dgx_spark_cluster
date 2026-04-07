# LLMの推論速度を比較
DGX Spark１台ではVRAMが不足する大規模言語モデルに対して、複数のDGX Sparkを使用して推論を行い、その処理速度を検証・比較する。
複数台のDGX Sparkを用いて仮想的な巨大GPUを構築するため、lamma.cppのRPC機能を利用する。

## 前準備と予備知識
ここでは、LLMの推論速度を比較するために、`llama-bench`を使用します。


## 推論速度比較結果

### 測定条件1
- モデル：Qwen3-235B-A22B Q2_K（79.80 GiB）
- pp128：128トークンのプロンプト処理速度（t/s）
- tg256：256トークンの生成速度（t/s）
- 測定回数：3回の平均（`-r 3`）

### 結果一覧1
| パターン | node数 | NW | backend | pp128 (t/s) | tg256 (t/s) |
|---|---:|---|---|---:|---:|
| 1台 | 1 | - | CUDA | 138.20 ± 9.54 | 18.47 ± 0.02 |
| 2台（QSFP） pair1 | 2 | QSFP 200Gbps | CUDA,RPC | 110.26 ± 5.50 | 16.50 ± 0.18 |
| 2台（QSFP） pair2 | 2 | QSFP 200Gbps | CUDA,RPC | 106.96 ± 5.57 | 16.18 ± 0.10 |
| 2台（RJ45） | 2 | RJ45 1Gbps | CUDA,RPC | 106.54 ± 7.20 | 16.30 ± 0.14 |
| 4台（RJ45） | 4 | RJ45 1Gbps | CUDA,RPC | - | - |
| 2台（RJ45） | 2 | RJ45 10Gbps | CUDA,RPC | - | - |
| 4台（RJ45） | 4 | RJ45 10Gbps | CUDA,RPC | - | - |


### 測定条件2
- モデル：Qwen3-235B-A22B Q2_K（79.80 GiB）
- pp8192：8192トークンのプロンプト処理速度（t/s）
- tg256：256トークンの生成速度（t/s）
- 測定回数：3回の平均（`-r 3`）

### 結果一覧2
| パターン | node数 | NW | backend | pp8192 (t/s) | tg256 (t/s) |
|---|---:|---|---|---:|---:|
| 1台 | 1 | - | CUDA |  |  |
| 2台（QSFP） pair1 | 2 | QSFP 200Gbps | CUDA,RPC | 191.53 ± 1.75 | 16.64 ± 0.05 |
| 2台（QSFP） pair2 | 2 | QSFP 200Gbps | CUDA,RPC |  |  |
| 2台（RJ45） | 2 | RJ45 1Gbps | CUDA,RPC | 169.79 ± 0.89 | 16.64 ± 0.05 |
| 4台（RJ45） | 4 | RJ45 1Gbps | CUDA,RPC | - | - |
| 2台（RJ45） | 2 | RJ45 10Gbps | CUDA,RPC | - | - |
| 4台（RJ45） | 4 | RJ45 10Gbps | CUDA,RPC | - | - |


### 測定条件3
- モデル：Qwen3-235B-A22B Q2_K（79.80 GiB）
- pp8192：8192トークンのプロンプト処理速度（t/s）
- tg2048：2048トークンの生成速度（t/s）
- 測定回数：3回の平均（`-r 3`）

### 結果一覧3
| パターン | node数 | NW | backend | pp8192 (t/s) | tg2048 (t/s) |
|---|---:|---|---|---:|---:|
| 1台 | 1 | - | CUDA |  |  |
| 2台（QSFP） pair1 | 2 | QSFP 200Gbps | CUDA,RPC | 195.53 ± 0.84 | 16.00 ± 0.02 |
| 2台（QSFP） pair2 | 2 | QSFP 200Gbps | CUDA,RPC |  |  |
| 2台（RJ45） | 2 | RJ45 1Gbps | CUDA,RPC | 169.81 ± 0.42 | 15.86 ± 0.02 |
| 4台（RJ45） | 4 | RJ45 1Gbps | CUDA,RPC | - | - |
| 2台（RJ45） | 2 | RJ45 10Gbps | CUDA,RPC | - | - |
| 4台（RJ45） | 4 | RJ45 10Gbps | CUDA,RPC | - | - |

## RPC対応コンテナの作成〜挙動確認
RPCに対応した仮想環境を作成します。
まず、以下のコマンドを実行してサンドボックスを作成し、RPC機能を追加を追加するためlamma.cppのクローンとビルドをしてください。
```
# サンドボックス作成
sudo singularity build --sandbox \
  /tmp/llama-sandbox \
  /home4cluster/containers/llama.cpp_full-cuda.sif

# cmake インストール
sudo singularity exec \
  --writable --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  /tmp/llama-sandbox \
  bash -c "apt-get update && apt-get install -y cmake build-essential"

# llama.cpp クローン＆ビルド
sudo singularity exec \
  --writable \
  --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  /tmp/llama-sandbox \
  bash -c "
    export PATH=/usr/local/cuda/bin:\$PATH && \
    rm -rf /build && \
    mkdir -p /build && cd /build && \
    git clone https://github.com/ggml-org/llama.cpp.git . && \
    mkdir build && cd build && \
    cmake .. \
      -DGGML_CUDA=ON \
      -DGGML_RPC=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_CUDA_ARCHITECTURES=121 && \
    cmake --build . --parallel 20 && \
    cp /build/build/bin/* /app/
  "
```
作成したサンドボックスを`.sif`に変換します。
```
sudo singularity build \
  /home4cluster/containers/llama-rpc.sif \
  /tmp/llama-sandbox
```
RPCバイナリを確認します。
```
singularity exec /home4cluster/containers/llama-rpc.sif \
  ls /app/ | grep rpc
# 出力：rpc-server 
```
最後に1台推論で動作を確認してください。
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

## ステップ2 : 2台推論（QSFP）
QSFPケーブルで接続された2台のDGX Sparkを使用して、推論を実行し、処理速度を測定します。
以下のコマンドでスクリプト作成し、実行してください。
```
# pair1 (node15-node16)
cat > /home4cluster/scripts/run_2node_qsfp_pair1.sh << 'EOF'
#!/bin/bash
# 2台推論テスト QSFP pair1 (node15↔node16)
# 管理者nodeから実行

SIF=/home4cluster/containers/llama-rpc.sif
MODEL=/home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf
LOG=/home4cluster/logs/2node_qsfp_pair1_$(date +%Y%m%d_%H%M%S).log
RPC_PORT=50052
NODE_MAIN=node15
NODE_WORKER=node16
WORKER_QSFP_IP=10.0.1.2  # node16 の QSFP IP

echo "=== 2台推論テスト QSFP pair1 (node15↔node16) ===" | tee $LOG
echo "開始: $(date)" | tee -a $LOG

# node16 で RPC サーバーを起動（QSFP IP でリッスン）
echo "RPCサーバー起動中（node16: ${WORKER_QSFP_IP}:${RPC_PORT}）..." | tee -a $LOG
ssh mprg@${NODE_WORKER} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/rpc-server --host ${WORKER_QSFP_IP} --port ${RPC_PORT}" &
SSH_PID=$!

# RPC サーバーの起動を待つ
sleep 15

# node15 から QSFP 経由で推論実行
echo "推論開始（node15 → node16 QSFP）..." | tee -a $LOG
ssh mprg@${NODE_MAIN} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/llama-bench \
  --model ${MODEL} \
  --n-gpu-layers 99 \
  --rpc ${WORKER_QSFP_IP}:${RPC_PORT} \
  -p 128 -n 256 -r 3" 2>&1 | tee -a $LOG

# 後片付け
echo "RPCサーバーを終了..." | tee -a $LOG
ssh mprg@${NODE_WORKER} "pkill -f rpc-server" 2>/dev/null
wait $SSH_PID 2>/dev/null
echo "終了: $(date)" | tee -a $LOG
EOF
chmod +x /home4cluster/scripts/run_2node_qsfp_pair1.sh

bash /home4cluster/scripts/run_2node_qsfp_pair1.sh
```
```
# pair2 (node17-node18)
cat > /home4cluster/scripts/run_2node_qsfp_pair2.sh << 'EOF'
#!/bin/bash
# 2台推論テスト QSFP pair2 (node17↔node18)
# 管理者nodeから実行

SIF=/home4cluster/containers/llama-rpc.sif
MODEL=/home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf
LOG=/home4cluster/logs/2node_qsfp_pair2_$(date +%Y%m%d_%H%M%S).log
RPC_PORT=50052
NODE_MAIN=node17
NODE_WORKER=node18
WORKER_QSFP_IP=10.0.2.2  # node18 の QSFP IP

echo "=== 2台推論テスト QSFP pair2 (node17↔node18) ===" | tee $LOG
echo "開始: $(date)" | tee -a $LOG

# node18 で RPC サーバーを起動（QSFP IP でリッスン）
echo "RPCサーバー起動中（node18: ${WORKER_QSFP_IP}:${RPC_PORT}）..." | tee -a $LOG
ssh mprg@${NODE_WORKER} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/rpc-server --host ${WORKER_QSFP_IP} --port ${RPC_PORT}" &
SSH_PID=$!

sleep 15

# node17 から QSFP 経由で推論実行
echo "推論開始（node17 → node18 QSFP）..." | tee -a $LOG
ssh mprg@${NODE_MAIN} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/llama-bench \
  --model ${MODEL} \
  --n-gpu-layers 99 \
  --rpc ${WORKER_QSFP_IP}:${RPC_PORT} \
  -p 128 -n 256 -r 3" 2>&1 | tee -a $LOG

# 後片付け
echo "RPCサーバーを終了..." | tee -a $LOG
ssh mprg@${NODE_WORKER} "pkill -f rpc-server" 2>/dev/null
wait $SSH_PID 2>/dev/null
echo "終了: $(date)" | tee -a $LOG
EOF
chmod +x /home4cluster/scripts/run_2node_qsfp_pair2.sh

bash /home4cluster/scripts/run_2node_qsfp_pair2.sh
```

以下が結果です（メモ書き。あとで表にする）。
```
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| qwen3moe 235B.A22B Q2_K - Medium |  79.80 GiB |   235.09 B | CUDA,RPC   |  99 |           pp128 |        110.26 ± 5.50 |
ggml_backend_cuda_get_available_uma_memory: final available_memory_kb: 73059416
ggml_backend_cuda_graph_compute: CUDA graph warmup complete
| qwen3moe 235B.A22B Q2_K - Medium |  79.80 GiB |   235.09 B | CUDA,RPC   |  99 |           tg256 |         16.50 ± 0.18 |
Client connection closed

build: 25eec6f32 (8672)
RPCサーバーを終了...
終了: Mon Apr  6 06:26:10 PM JST 2026
```

## ステップ3 : 2台推論（RJ45）
```
cat > /home4cluster/scripts/run_2node_rj45_1g.sh << 'EOF'
#!/bin/bash
# 2台推論テスト RJ45 1G (node15↔node16)
# 管理者nodeから実行

SIF=/home4cluster/containers/llama-rpc.sif
MODEL=/home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf
LOG=/home4cluster/logs/2node_rj45_1g_$(date +%Y%m%d_%H%M%S).log
RPC_PORT=50052
NODE_MAIN=node15
NODE_WORKER=node16
WORKER_RJ45_IP=10.0.0.16  # node16 の RJ45 IP

echo "=== 2台推論テスト RJ45 1G (node15↔node16) ===" | tee $LOG
echo "開始: $(date)" | tee -a $LOG

# node16 で RPC サーバーを起動（RJ45 IP でリッスン）
echo "RPCサーバー起動中（node16: ${WORKER_RJ45_IP}:${RPC_PORT}）..." | tee -a $LOG
ssh mprg@${NODE_WORKER} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/rpc-server --host ${WORKER_RJ45_IP} --port ${RPC_PORT}" &
SSH_PID=$!

sleep 15

# node15 から RJ45 経由で推論実行
echo "推論開始（node15 → node16 RJ45 1G）..." | tee -a $LOG
ssh mprg@${NODE_MAIN} \
  "singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  ${SIF} \
  /app/llama-bench \
  --model ${MODEL} \
  --n-gpu-layers 99 \
  --rpc ${WORKER_RJ45_IP}:${RPC_PORT} \
  -p 128 -n 256 -r 3" 2>&1 | tee -a $LOG

# 後片付け
echo "RPCサーバーを終了..." | tee -a $LOG
ssh mprg@${NODE_WORKER} "pkill -f rpc-server" 2>/dev/null
wait $SSH_PID 2>/dev/null
echo "終了: $(date)" | tee -a $LOG
EOF
chmod +x /home4cluster/scripts/run_2node_rj45_1g.sh
```
```
bash /home4cluster/scripts/run_2node_rj45_1g.sh
```

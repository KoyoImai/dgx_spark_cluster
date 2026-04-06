# DGX Spark4台を使用した言語モデルの利用
構築したDGX Spark4台の環境を使って大規模言語モデルを動かします。
使用するDGX Sparkの台数、QSFP、RJ45の差によって推論速度などにどのような影響が出るかを確かめます。
検証パターンをいかにまとめます。

| パターン | partition | node数 | NW | 備考 |
|---|---|---:|---|---|
| 1台 | pair1 | 1 | - | node15固定 |
| 2台（QSFP） | pair1 | 2 | QSFP 200Gbps | node15↔16 |
| 2台（QSFP） | pair2 | 2 | QSFP 200Gbps | node17↔18 |
| 2台（RJ45） | all | 2 | RJ45 1Gbps | node15↔16（同構成でNW変更） |
| 4台（RJ45） | all | 4 | RJ45 1Gbps | node15〜18 |
| 2台（RJ45） | all | 2 | RJ45 10Gbps | node15↔16（同構成でNW変更） |
| 4台（RJ45） | all | 4 | RJ45 10Gbps | node15〜18 |

## ステップ1：Singularity環境の用意
ここでは、クラスタ上で大規模言語モデルを動かすための環境を用意します。

まず、管理者nodeで以下のコマンドを実行してください。
```
cd /home4cluster/
sudo mkdir containers
cd containers/
sudo chmod 777 /home4cluster/containers
singularity pull docker://ghcr.io/ggml-org/llama.cpp:full-cuda
```
node15でSlurn jobとして実行して確認します。
```
srun --partition=pair1 --nodes=1 --gres=gpu:1 \
  singularity exec --nv /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi
```
また、4台のDGX Sparkで同時にGPUが使用可能かを確かめます。
以下のコマンドを実行して結果を確認してください。
```
mprg@spark-3894:/home4cluster/containers$ srun --partition=all --nodes=4 --gres=gpu:1 \
  singularity exec --nv \
  --env LD_LIBRARY_PATH=/app \
  /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi --query-gpu=name,memory.total,memory.free --format=csv
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
mprg@spark-3894:/home4cluster/containers$ srun --partition=all --nodes=4 --gres=gpu:1 \
  singularity exec --nv \
  --env LD_LIBRARY_PATH=/app \
  /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0  On |                  N/A |
| N/A   38C    P8              5W /  N/A  | Not Supported          |      1%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3821      G   /usr/lib/xorg/Xorg                      166MiB |
|    0   N/A  N/A            3963      G   /usr/bin/gnome-shell                    179MiB |
|    0   N/A  N/A            4862      G   .../7965/usr/lib/firefox/firefox        587MiB |
|    0   N/A  N/A           12599      G   /usr/bin/gnome-control-center            32MiB |
+-----------------------------------------------------------------------------------------+
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   36C    P8              4W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   35C    P8              3W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3338      G   /usr/lib/xorg/Xorg                      133MiB |
|    0   N/A  N/A            3491      G   /usr/bin/gnome-shell                     92MiB |
|    0   N/A  N/A            4415      G   .../7965/usr/lib/firefox/firefox        695MiB |
|    0   N/A  N/A            9715      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
|    0   N/A  N/A            3439      G   /usr/lib/xorg/Xorg                      111MiB |
|    0   N/A  N/A            3592      G   /usr/bin/gnome-shell                     90MiB |
|    0   N/A  N/A            4409      G   .../7965/usr/lib/firefox/firefox        490MiB |
|    0   N/A  N/A           10172      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   35C    P8              4W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3644      G   /usr/lib/xorg/Xorg                      111MiB |
|    0   N/A  N/A            3787      G   /usr/bin/gnome-shell                     78MiB |
|    0   N/A  N/A            4333      G   ...exec/xdg-desktop-portal-gnome         76MiB |
|    0   N/A  N/A            4603      G   .../7965/usr/lib/firefox/firefox        479MiB |
|    0   N/A  N/A           11820      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
mprg@spark-3894:/home4cluster/containers$ 
```

## ステップ2：学習済み大規模言語モデルの用意
huggingfaceから学習済みパラメータをダウンロードしてきます。
以下のコマンドを実行してください。
```
cd /home4cluster/models

# ファイル1（49.9GB）
wget "https://huggingface.co/unsloth/Qwen3-235B-A22B-GGUF/resolve/main/Q2_K/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf"

# ファイル2（35.8GB）
wget "https://huggingface.co/unsloth/Qwen3-235B-A22B-GGUF/resolve/main/Q2_K/Qwen3-235B-A22B-Q2_K-00002-of-00002.gguf"
```


## ステップ3 : 大規模言語モデルで推論を実行
以下のコマンドを実行して、推論可能かを確かめてください。
1nodeだけの確認実行です。
```
srun --partition=pair1 --nodes=1 --nodelist=node15 --gres=gpu:1 \
  singularity exec --nv \
  --env LD_LIBRARY_PATH=/app \
  /home4cluster/containers/llama.cpp_full-cuda.sif \
  /app/llama-completion \
  --model /home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf \
  --n-gpu-layers 99 \
  --ctx-size 4096 \
  --temp 0.6 \
  --top-p 0.95 \
  --top-k 20 \
  --seed 3407 \
  --no-conversation \
  --prompt "日本語で自己紹介してください。" \
  -n 256
```

## ステップ4 : クラスタ環境の整備（一旦不要）
**[参考1:llama.cpp(1)](https://zenn.dev/onpremdev/articles/3d77c91717d61e)**

全ての実行を管理者nodeから行えるようにクラスタ環境を整備します。
まず、管理者nodeでSSH鍵を生成し、それを全計算nodeにコピーします。
以下のコマンドを実行してください。
```
# mgmtで実行
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# 全計算ノードに公開鍵をコピー
ssh-copy-id mprg@node15
ssh-copy-id mprg@node16
ssh-copy-id mprg@node17
ssh-copy-id mprg@node18
```

## ステップ5 : スクリプトの作成と実行
**[参考1:llama.cpp(1)](https://zenn.dev/onpremdev/articles/3d77c91717d61e)**

### 1nodeで実行
まず、1nodeでの実行を行います。
以下のコマンドでスクリプトを作成してください。
```
cat > /home4cluster/scripts/run_1node.sh << 'EOF'
#!/bin/bash
# 1台推論テスト（node15）

SIF=/home4cluster/containers/llama.cpp_full-cuda.sif
MODEL=/home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf
LOG=/home4cluster/logs/1node_$(date +%Y%m%d_%H%M%S).log

echo "=== 1台推論テスト (node15) ===" | tee $LOG
echo "開始: $(date)" | tee -a $LOG

singularity exec --nv --env LD_LIBRARY_PATH=/app $SIF \
  /app/llama-bench \
  --model $MODEL \
  --n-gpu-layers 99 \
  -p 128 -n 256 -r 3 2>&1 | tee -a $LOG

echo "終了: $(date)" | tee -a $LOG
EOF
```
スクリプトを作成したら実行してください。
```
mprg@spark-3894:/home4cluster$ bash /home4cluster/scripts/run_1node.sh
=== 1台推論テスト (node15) ===
開始: Sat Apr  4 03:20:44 PM JST 2026
ggml_cuda_init: found 1 CUDA devices (Total VRAM: 122572 MiB):
  Device 0: NVIDIA GB10, compute capability 12.1, VMM: yes, VRAM: 122572 MiB
load_backend: loaded CUDA backend from /app/libggml-cuda.so
load_backend: loaded CPU backend from /app/libggml-cpu-armv8.6_2.so
| model                          |       size |     params | backend    | ngl |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | --------------: | -------------------: |
| qwen3moe 235B.A22B Q2_K - Medium |  79.80 GiB |   235.09 B | CUDA       |  99 |           pp128 |        134.11 ± 7.43 |
| qwen3moe 235B.A22B Q2_K - Medium |  79.80 GiB |   235.09 B | CUDA       |  99 |           tg256 |         17.54 ± 0.08 |

build: f49e91787 (8643)
終了: Sat Apr  4 03:27:49 PM JST 2026
mprg@spark-3894:/home4cluster$
```


### 2nodeで実行
続いて、2node-rj45での実行を行います。
2node以上で実行するには、`lamma.cpp rpc`が必要であり、現在使用しているsifファイルは`lamma.cpp rpc`が入っていないので、新しく作り直します。
以下のコマンドでDockerファイルを作成し、イメージをビルドしてください。
```
sudo mkdir -p /home4cluster/docker/llama-rpc
sudo cat > /home4cluster/docker/llama-rpc/Dockerfile << 'EOF'
FROM ghcr.io/ggml-org/llama.cpp:full-cuda AS base

# llama.cppをRPC有効で再ビルド
RUN apt-get update && apt-get install -y git cmake build-essential

WORKDIR /build
RUN git clone https://github.com/ggml-org/llama.cpp.git . && \
    mkdir build && cd build && \
    cmake .. -DGGML_CUDA=ON -DGGML_RPC=ON -DCMAKE_BUILD_TYPE=Release && \
    cmake --build . --parallel 20

# ビルドした実行ファイルを/appにコピー
RUN cp /build/build/bin/* /app/ 2>/dev/null || true
EOF

cd /home4cluster/docker/llama-rpc
docker build -t llama-cpp-rpc .

```
以下のコマンドでスクリプトを作成してください。
```
cat > /home4cluster/scripts/run_2node_qsfp_pair1.sh << 'EOF'
#!/bin/bash
# 2台推論テスト QSFP（pair1: node15↔16）

SIF=/home4cluster/containers/llama.cpp_full-cuda.sif
MODEL=/home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf
LOG=/home4cluster/logs/2node_qsfp_pair1_$(date +%Y%m%d_%H%M%S).log
RPC_PORT=50052
WORKER=node16
WORKER_IP=10.0.1.2  # node16 QSFP IP

echo "=== 2台推論テスト QSFP pair1 (node15↔node16) ===" | tee $LOG
echo "開始: $(date)" | tee -a $LOG

# node16でRPCサーバーを起動
ssh mprg@$WORKER \
  "singularity exec --nv --env LD_LIBRARY_PATH=/app $SIF \
  /app/rpc-server --host $WORKER_IP --port $RPC_PORT" &
SSH_PID=$!
echo "RPCサーバー起動中（node16: $WORKER_IP:$RPC_PORT）..." | tee -a $LOG
sleep 15

# node15から推論実行
singularity exec --nv --env LD_LIBRARY_PATH=/app $SIF \
  /app/llama-bench \
  --model $MODEL \
  --n-gpu-layers 99 \
  --rpc ${WORKER_IP}:${RPC_PORT} \
  -p 128 -n 256 -r 3 2>&1 | tee -a $LOG

# 後片付け
echo "RPCサーバーを終了..." | tee -a $LOG
ssh mprg@$WORKER "pkill -f rpc-server" 2>/dev/null
wait $SSH_PID 2>/dev/null
echo "終了: $(date)" | tee -a $LOG
EOF
```
作成したスクリプトを実行して結果を確認してください。
```

```

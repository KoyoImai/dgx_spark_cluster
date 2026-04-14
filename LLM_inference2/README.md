# LLMの推論速度を比較2
LLM_inferenc edえは，lamma.cpp RPCを用いた場合において，lamma-benchで評価した．
ここでは，vLLM+Rayによる推論速度などの評価を行う．

**[参考1:vLLM](https://zenn.dev/karaage0703/articles/fcca40c614dffd)** 

**[参考2:vLLM](https://licensecounter.jp/engineer-voice/blog/articles/20251219_dgx_spark_4vllm.html)**

**[参考3:vLLM](https://zenn.dev/t0d4/articles/7d8951b7c40fba)**

**[参考4:vLLM](https://zenn.dev/munakatakm/articles/106556cbbd75d6)**


## 比較表
### 測定条件
- モデル：GPT-OSS-120B（MXFP4量子化）
- 推論エンジン：vLLM 0.11.0（nvcr.io/nvidia/vllm:25.11-py3）
- データセット：ShareGPT
- ベンチマークツール：vllm bench serve
- All Reduce：Tensor Parallelism
- seed：42
- warmup：1

### 結果一覧

| パターン | node数 | NW | num-prompts | Output tok/s | Peak Output tok/s | TTFT平均値 (ms) | TTFT中央値 (ms) | P99TTFT (ms) | TPOT平均 (ms) | TPOT中央値 (ms) | P99TPOT (ms) |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1台 | 1 | - | 1 | 32.28 | 33.00 | 71.73 | 71.73 | 71.73 | 30.64 | 30.64 | 30.64 |
| 1台 | 1 | - | 10 | 83.11 | 117.00 | 244.92 | 263.32 | 265.92 | 66.76 | 67.26 | 76.60 |
| 1台 | 1 | - | 100 | 194.24 | 399.00 | 623.93 | 486.99 | 933.90 | 227.88 | 235.00 | 287.34 |
| 1台 | 1 | - | 250 | 288.72 | 750.00 | 1221.08 | 1327.43 | 1337.46 | 345.71 | 354.47 | 423.84 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 1 | 31.25 | 32.00 | 67.84 | 67.84 | 67.84 | 31.70 | 31.70 | 31.70 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 10 | 91.04 | 144.00 | 198.95 | 236.96 | 237.74 | 59.67 | 61.25 | 67.82 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 100 | 239.41 | 499.00 | 904.21 | 919.67 | 928.12 | 178.16 | 185.44 | 205.72 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 250 | 349.92 | 751.00 | 1187.19 | 1432.73 | 1439.06 | 290.06 | 291.99 | 419.12 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 1 | 31.79 | 32.00 | 59.52 | 59.52 | 59.52 | 31.22 | 31.22 | 31.22 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 10 | 84.59 | 135.00 | 194.51 | 208.19 | 209.87 | 60.25 | 60.19 | 68.04 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 100 | 242.94 | 515.00 | 619.19 | 501.22 | 865.38 | 172.30 | 171.39 | 231.21 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 250 | 369.21 | 921.00 | 1364.93 | 1646.18 | 1657.20 | 266.39 | 265.40 | 413.92 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 1 | 14.21 | 16.00 | 103.43 | 103.43 | 103.43 | 70.10 | 70.10 | 70.10 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 10 | 56.68 | 108.00 | 209.96 | 223.61 | 225.43 | 84.57 | 85.75 | 89.16 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 100 | 208.63 | 594.00 | 635.56 | 521.82 | 876.06 | 182.73 | 192.24 | 213.71 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 250 | 344.57 | 972.00 | 1615.74 | 1809.02 | 1816.66 | 255.07 | 245.11 | 379.78 |

## ステップ1：Dockerの再インストール
Singularityインストール時に，ryncの依存関係問題でdocker-ceが削除されてしまっているので，Dockerを再インストールします．
以下のコマンドを実行して下さい．
```
sudo apt-get update
sudo apt-get install -y docker-ce
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```

## ステップ2:Docker Imageのpull
Dockerの再インストールが完了したら，Docker Imageをpullします．
以下のコマンドを実行して下さい．
```
docker pull nvcr.io/nvidia/vllm:25.11-py3
```

## ステップ3:run_cluster.shの取得と修正
管理者nodeで`run_cluster.sh`を取得し，内容を修正します．
以下のコマンドを実行して下さい．
```
cd /home4cluster/

# スクリプトをダウンロード
wget https://raw.githubusercontent.com/vllm-project/vllm/refs/heads/main/examples/online_serving/run_cluster.sh

# headノードIPを引数で指定できるよう修正
sed -i 's/RAY_START_CMD+=" --head --port=6379"/RAY_START_CMD+=" --head --node-ip-address=${HEAD_NODE_ADDRESS} --port=6379"/' run_cluster.sh

chmod +x run_cluster.sh

# 修正が正しく入っているか確認
grep "node-ip-address" run_cluster.sh
```

## ステップ4:事前学習済みパラメータの取得
事前学習済みパラメータをHugging Faceから取得します．
以下のコマンドを実行して下さい．
```
docker run --rm \
  -v /home4cluster/models/hf:/models \
  -e HF_HOME=/models/.cache \
  nvcr.io/nvidia/vllm:25.11-py3 \
  huggingface-cli download openai/gpt-oss-120b \
  --local-dir /models/gpt-oss-120b \
  --local-dir-use-symlinks False
```

## ステップ5:1nodeでの動作確認
1nodeで動作を確認します．
まず，以下のコマンドを実行して推論サーバーを起動してください．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster/models/hf:/models \
  nvcr.io/nvidia/vllm:25.11-py3 \
  vllm serve /models/gpt-oss-120b \
    --host 0.0.0.0 \
    --port 8000 \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.90 \
    --enforce-eager \
    --enable-expert-parallel \
    --tool-call-parser openai \
    --enable-auto-tool-choice
```
推論サーバーを起動したら，以下のコマンドを実行して推論を実行して下さい．
```
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/models/gpt-oss-120b",
    "messages": [{"role": "user", "content": "日本語で自己紹介してください。"}],
    "max_tokens": 512
  }'
```

## ステップ6:node1つでベンチマーク
**[参考](https://forums.developer.nvidia.com/t/6x-spark-setup/354399)**

まず，ベンチマーク用データセットを準備します．
以下のコマンドを管理者nodeで実行して，ShareGPTデータセットの準備をしてください．
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  wget -O /home4cluster/ShareGPT_V3_unfiltered_cleaned_split.json \
  https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
```
データセットの準備が完了したら，vLLMサーバーを起動します．
ここでは，node15でベンチマーク評価を行います．
node15で以下のコマンドを実行して，vLLMサーバーを起動して下さい．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster/models/hf:/models \
  nvcr.io/nvidia/vllm:25.11-py3 \
  vllm serve /models/gpt-oss-120b \
    --host 0.0.0.0 \
    --port 8000 \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.90 \
    --enforce-eager \
    --enable-expert-parallel \
    --tool-call-parser openai \
    --enable-auto-tool-choice \
    --served-model-name openai/gpt-oss-120b
```
サーバー起動後，別のターミナルを起動し，node15で以下のコマンドを実行してください．
```
for NUM_PROMPTS in 1 10 100 250; do
  echo "=== num-prompts=${NUM_PROMPTS} ===" | tee -a /home4cluster/logs/vllm/1node_$(date +%Y%m%d).log
  docker run --rm \
    --gpus all \
    --network host \
    -v /home4cluster:/home4cluster \
    nvcr.io/nvidia/vllm:25.11-py3 \
    vllm bench serve \
      --backend vllm \
      --model openai/gpt-oss-120b \
      --host localhost \
      --port 8000 \
      --endpoint /v1/completions \
      --dataset-name sharegpt \
      --dataset-path /home4cluster/ShareGPT_V3_unfiltered_cleaned_split.json \
      --seed 42 \
      --num-prompts ${NUM_PROMPTS} \
    2>&1 | tee -a /home4cluster/logs/vllm/1node_$(date +%Y%m%d).log
  sleep 5
done
```

## ステップ7:2nodeでのベンチマーク（QSFP）
QSFPでやっても速度が向上しない場合，以下をコマンドを実行します．
```
# 現在の設定確認
sysctl net.core.rmem_max
sysctl net.core.wmem_max
sysctl net.ipv4.tcp_rmem
sysctl net.ipv4.tcp_wmem

# チューニング例（BDP以上に設定）
sudo sysctl -w net.core.rmem_max=67108864
sudo sysctl -w net.core.wmem_max=67108864

# 送信バッファも拡張する
sudo sysctl -w net.ipv4.tcp_wmem="4096 16384 67108864"
sudo sysctl -w net.ipv4.tcp_rmem="4096 131072 67108864"
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

```

node15とnode16の2node構成でベンチマークを評価します．
まず，node15（head）で以下のコマンドを実行して，Rayクラスタを起動してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.1.1

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --head \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${HEAD_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
その後，node16（worker）で以下のコマンドを実行し，クラスタに参加してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.1.1
export WORKER_IP=10.0.1.2

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
クラスタが組めているかを以下のコマンドで確認して下さい．
```
mprg@spark-fb97:~$ docker ps
CONTAINER ID   IMAGE                           COMMAND                   CREATED              STATUS              PORTS     NAMES
1da3039c0282   nvcr.io/nvidia/vllm:25.11-py3   "/bin/bash -c 'ray s…"   About a minute ago   Up About a minute             node-58
mprg@spark-fb97:~$ docker exec node-58 ray status
======== Autoscaler status: 2026-04-10 05:23:31.894412 ========
Node status
---------------------------------------------------------------
Active:
 1 node_a81cfbad5c01fb9a0060b29a3698e0ec84f5ec6b0349790e329b7410
 1 node_4e6d26d7cb77af6e44029cf01b66e161eb5b03a9ecbd79ed6117ee36
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/40.0 CPU
 0.0/2.0 GPU
 0B/218.83GiB memory
 0B/19.46GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
```
node15とnode16の2node構成でクラスタが構築できていることを確認したら，1nodeのときと同様に，vLLMサーバーを起動します．
以下のコマンドを実行して下さい．
```
vllm serve /root/.cache/huggingface/gpt-oss-120b \
  --tensor-parallel-size 2 \
  --enable-expert-parallel \
  --host 0.0.0.0 \
  --port 8000 \
  --gpu-memory-utilization 0.89 \
  --enforce-eager \
  --tool-call-parser openai \
  --enable-auto-tool-choice \
  --served-model-name openai/gpt-oss-120b
```
vLLMサーバー起動後，以下のコマンドで評価を実行して下さい．
```
for NUM_PROMPTS in 1 10 100 250; do
  echo "=== num-prompts=${NUM_PROMPTS} ===" | tee -a /home4cluster/logs/vllm/2node_qsfp_pair1_$(date +%Y%m%d).log
  docker run --rm \
    --gpus all \
    --network host \
    -v /home4cluster:/home4cluster \
    nvcr.io/nvidia/vllm:25.11-py3 \
    vllm bench serve \
      --backend vllm \
      --model openai/gpt-oss-120b \
      --host localhost \
      --port 8000 \
      --endpoint /v1/completions \
      --dataset-name sharegpt \
      --dataset-path /home4cluster/ShareGPT_V3_unfiltered_cleaned_split.json \
      --seed 42 \
      --num-prompts ${NUM_PROMPTS} \
    2>&1 | tee -a /home4cluster/logs/vllm/2node_qsfp_pair1_$(date +%Y%m%d).log
  sleep 5
done
```

## ステップ8:2nodeでのベンチマーク（RJ45）
node15とnode16の2node構成でベンチマークを評価します．
まず，node15（head）で以下のコマンドを実行して，Rayクラスタを起動してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --head \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${HEAD_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
その後，node16（worker）で以下のコマンドを実行し，クラスタに参加してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15
export WORKER_IP=10.0.0.16

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
node15とnode16の2node構成でクラスタが構築できていることを確認したら，1nodeのときと同様に，vLLMサーバーを起動します．
以下のコマンドを実行して下さい．
```
# コンテナに入る
docker exec -it node-9745 /bin/bash

# vLLMサーバー起動
vllm serve /root/.cache/huggingface/gpt-oss-120b \
  --tensor-parallel-size 2 \
  --enable-expert-parallel \
  --host 0.0.0.0 \
  --port 8000 \
  --gpu-memory-utilization 0.89 \
  --enforce-eager \
  --tool-call-parser openai \
  --enable-auto-tool-choice \
  --served-model-name openai/gpt-oss-120b
```

```
for NUM_PROMPTS in 1 10 100 250; do
  echo "=== num-prompts=${NUM_PROMPTS} ===" | tee -a /home4cluster/logs/vllm/2node_rj45_1g_$(date +%Y%m%d).log
  docker run --rm \
    --gpus all \
    --network host \
    -v /home4cluster:/home4cluster \
    nvcr.io/nvidia/vllm:25.11-py3 \
    vllm bench serve \
      --backend vllm \
      --model openai/gpt-oss-120b \
      --host localhost \
      --port 8000 \
      --endpoint /v1/completions \
      --dataset-name sharegpt \
      --dataset-path /home4cluster/ShareGPT_V3_unfiltered_cleaned_split.json \
      --seed 42 \
      --num-prompts ${NUM_PROMPTS} \
    2>&1 | tee -a /home4cluster/logs/vllm/2node_rj45_1g_$(date +%Y%m%d).log
  sleep 5
done
```


## ステップ9:4nodeでのベンチマーク（RJ45）
node15~node18の4node構成でベンチマークを評価します．
まず，node15（head）で以下のコマンドを実行して，Rayクラスタを起動してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --head \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${HEAD_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
その後，node16（worker）で以下のコマンドを実行し，クラスタに参加してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15
export WORKER_IP=10.0.0.16

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
その後，node17（worker）で以下のコマンドを実行し，クラスタに参加してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15
export WORKER_IP=10.0.0.17

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
その後，node18（worker）で以下のコマンドを実行し，クラスタに参加してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enP7s7
export HEAD_IP=10.0.0.15
export WORKER_IP=10.0.0.18

bash /home4cluster/run_cluster.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e OMPI_MCA_btl_tcp_if_include=${MN_IF_NAME} \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
node15~node18の4node構成でクラスタが構築できていることを確認したら，1nodeのときと同様に，vLLMサーバーを起動します．
以下のコマンドを実行して下さい．
```
# コンテナに入る
docker exec -it node-28328 /bin/bash

# vLLMサーバー起動
vllm serve /root/.cache/huggingface/gpt-oss-120b \
  --tensor-parallel-size 4 \
  --enable-expert-parallel \
  --host 0.0.0.0 \
  --port 8000 \
  --gpu-memory-utilization 0.89 \
  --enforce-eager \
  --tool-call-parser openai \
  --enable-auto-tool-choice \
  --served-model-name openai/gpt-oss-120b
```

```
for NUM_PROMPTS in 1 10 100 250; do
  echo "=== num-prompts=${NUM_PROMPTS} ===" | tee -a /home4cluster/logs/vllm/4node_rj45_1g_$(date +%Y%m%d).log
  docker run --rm \
    --gpus all \
    --network host \
    -v /home4cluster:/home4cluster \
    nvcr.io/nvidia/vllm:25.11-py3 \
    vllm bench serve \
      --backend vllm \
      --model openai/gpt-oss-120b \
      --host localhost \
      --port 8000 \
      --endpoint /v1/completions \
      --dataset-name sharegpt \
      --dataset-path /home4cluster/ShareGPT_V3_unfiltered_cleaned_split.json \
      --seed 42 \
      --num-prompts ${NUM_PROMPTS} \
    2>&1 | tee -a /home4cluster/logs/vllm/4node_rj45_1g_$(date +%Y%m%d).log
  sleep 5
done
```


# LLMの推論速度評価
以下のように，QSFP接続した状態を1つのnodeとして捉えて，LLMの推論を行う．
```
[node15+16] → vLLMサーバー (port 8000) ┐
                                         → ロードバランサー → ベンチマーク
[node17+18] → vLLMサーバー (port 8000) ┘
```

## 結果比較

| パターン | node数 | NW | num-prompts | Output tok/s | Peak Output tok/s | TTFT平均値 (ms) | TTFT中央値 (ms) | P99TTFT (ms) | TPOT平均 (ms) | TPOT中央値 (ms) | P99TPOT (ms) |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1台 | 1 | - | 1 | 32.28 | 33.00 | 71.73 | 71.73 | 71.73 | 30.64 | 30.64 | 30.64 |
| 1台 | 1 | - | 10 | 83.11 | 117.00 | 244.92 | 263.32 | 265.92 | 66.76 | 67.26 | 76.60 |
| 1台 | 1 | - | 100 | 194.24 | 399.00 | 623.93 | 486.99 | 933.90 | 227.88 | 235.00 | 287.34 |
| 1台 | 1 | - | 250 | 288.72 | 750.00 | 1221.08 | 1327.43 | 1337.46 | 345.71 | 354.47 | 423.84 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 1 | 44.36 | 45.00 | 47.96 | 47.96 | 47.96 | 22.33 | 22.33 | 22.33 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 10 | 120.78 | 189.00 | 138.49 | 102.87 | 192.78 | 43.99 | 45.31 | 49.69 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 100 | 301.50 | 663.00 | 448.47 | 486.93 | 494.62 | 149.47 | 148.22 | 207.26 |
| 2台（QSFP）pair1 | 2 | QSFP 200Gbps | 250 | 468.88 | 1482.00 | 784.15 | 887.45 | 900.00 | 197.65 | 195.89 | 260.24 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 1 | 31.79 | 32.00 | 59.52 | 59.52 | 59.52 | 31.22 | 31.22 | 31.22 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 10 | 84.59 | 135.00 | 194.51 | 208.19 | 209.87 | 60.25 | 60.19 | 68.04 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 100 | 242.94 | 515.00 | 619.19 | 501.22 | 865.38 | 172.30 | 171.39 | 231.21 |
| 2台（RJ45）1Gbps | 2 | RJ45 1Gbps | 250 | 369.21 | 921.00 | 1364.93 | 1646.18 | 1657.20 | 266.39 | 265.40 | 413.92 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 1 | 14.21 | 16.00 | 103.43 | 103.43 | 103.43 | 70.10 | 70.10 | 70.10 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 10 | 56.68 | 108.00 | 209.96 | 223.61 | 225.43 | 84.57 | 85.75 | 89.16 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 100 | 208.63 | 594.00 | 635.56 | 521.82 | 876.06 | 182.73 | 192.24 | 213.71 |
| 4台（RJ45）1Gbps | 4 | RJ45 1Gbps | 250 | 344.57 | 972.00 | 1615.74 | 1809.02 | 1816.66 | 255.07 | 245.11 | 379.78 |
| 2台(15-16nodeと17-18node) | 1 | - | 1 | 44.35 | 45.00 | 53.16 | 53.16 | 53.16 | 22.29 | 22.29 | 22.29 |
| 2台(15-16nodeと17-18node) | 1 | - | 10 | 155.72 | 251.00 | 124.39 | 129.37 | 166.05 | 33.36 | 33.90 | 37.73 |
| 2台(15-16nodeと17-18node) | 1 | - | 100 | 400.55 | 889.00 | 421.85 | 407.51 | 603.65 | 102.95 | 104.92 | 152.87 |
| 2台(15-16nodeと17-18node) | 1 | - | 250 | 619.43 | 1385.00 | 1423.09 | 1349.61 | 2831.15 | 168.36 | 161.70 | 326.85 |

## 事前準備
### ステップ1:ペア1（node15+node16）でRayクラスタを起動
まず，ペア1でRayクラスタとvLLMサーバーを起動します．
以下のコマンドをnode15（head）で実行してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.1.1

bash /home4cluster/run_cluster_qsfp.sh ${VLLM_IMAGE} ${HEAD_IP} --head \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${HEAD_IP} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_IB_DISABLE=0 \
  -e NCCL_NET_GDR_LEVEL=5 \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
続いて，node16（worker）で以下のコマンドを実行してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.1.1
export WORKER_IP=10.0.1.2

bash /home4cluster/run_cluster_qsfp.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_IB_DISABLE=0 \
  -e NCCL_NET_GDR_LEVEL=5 \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
これで，node15とnode16でRayクラスタの起動が完了しました．
以下のコマンドで，Rayクラスタが起動できているかを確認してください．
```
mprg@spark-fb97:~$ docker ps
CONTAINER ID   IMAGE                           COMMAND                   CREATED          STATUS          PORTS     NAMES
0531c5c2f60e   nvcr.io/nvidia/vllm:25.11-py3   "/bin/bash -c 'ray s…"   23 seconds ago   Up 23 seconds             node-26018
mprg@spark-fb97:~$ docker exec node-26018 ray status
======== Autoscaler status: 2026-04-16 03:57:11.622696 ========
Node status
---------------------------------------------------------------
Active:
 1 node_adc07b94671c73d96f502962459d1f516a779e3808377fbde1e5e1b2
 1 node_09617c712f01827029c992ee84fd56531d7f205b7a25c704cdadb6cf
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/40.0 CPU
 0.0/2.0 GPU
 0B/166.80GiB memory
 0B/71.48GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
mprg@spark-fb97:~$ 
```

### ステップ2：nginxによるロードバランサーの設定
管理者node8で，nginxの設定を行います．
まず，nginxをインストールするために，以下のコマンドを実行してください．
```
sudo apt install -y nginx
```
続いて，必要な設定ファイルを要します．
`/etc/nginx/conf.d/vllm_lb.conf`として，以下の内容を追加してください．
```
upstream vllm_backends {
    server 10.0.1.1:8000;  # pair1 (node15)
    server 10.0.2.1:8000;  # pair2 (node17)
}

server {
    listen 9000;

    location / {
        proxy_pass http://vllm_backends;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```
最後に，nginxを起動するため，以下のコマンドを実行します．
```
sudo nginx -t
sudo systemctl restart nginx
```

### ステップ3:ペア2（node17+node18）でRayクラスタを起動
ステップ1と同様の手順で，ペア2でもRayクラスタとvLLMサーバーを起動します．
node17（head）で以下のコマンドを実行してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.2.1

bash /home4cluster/run_cluster_qsfp.sh ${VLLM_IMAGE} ${HEAD_IP} --head \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${HEAD_IP} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_IB_DISABLE=0 \
  -e NCCL_NET_GDR_LEVEL=5 \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```
nod18（worker）で以下のコマンドを実行してください．
```
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
export MN_IF_NAME=enp1s0f0np0
export HEAD_IP=10.0.2.1
export WORKER_IP=10.0.2.2

bash /home4cluster/run_cluster_qsfp.sh ${VLLM_IMAGE} ${HEAD_IP} --worker \
  /home4cluster/models/hf \
  -e VLLM_HOST_IP=${WORKER_IP} \
  -e NCCL_SOCKET_IFNAME=${MN_IF_NAME} \
  -e UCX_NET_DEVICES=${MN_IF_NAME} \
  -e NCCL_IB_DISABLE=0 \
  -e NCCL_NET_GDR_LEVEL=5 \
  -e GLOO_SOCKET_IFNAME=${MN_IF_NAME} \
  -e TP_SOCKET_IFNAME=${MN_IF_NAME} \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=${HEAD_IP}
```


### ステップ4：vLLMサーバーの起動
まず，pair1（node15とnode16）でvLLMサーバーを起動します．
node15のコンテナ内で以下のコマンドを実行してください．
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
続いて，pair2（node17とnode18）でもvLLMサーバーを起動します．
以下のコマンドを，node17のコンテナ内で実行して下さい．
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



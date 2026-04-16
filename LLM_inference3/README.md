# LLMの推論速度評価
以下のように，QSFP接続した状態を1つのnodeとして捉えて，LLMの推論を行う．
```
[node15+16] → vLLMサーバー (port 8000) ┐
                                         → ロードバランサー → ベンチマーク
[node17+18] → vLLMサーバー (port 8000) ┘
```

## 事前準備
### ステップ1:ペア1（node15+node16）で「RayクラスタとvLLMサーバーを起動
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







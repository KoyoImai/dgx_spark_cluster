# 大規模言語モデルの学習
大規模言語モデルを複数台のDGX-Sparkを使用して学習し，その速度を評価する．
学習速度の評価にはnanochatを使用する．

## ステップ1:nanochatのセットアップ
まず始めに，nanochatのセットアップを行う．
以下のコマンドを実行し，nanochatのリポジトリをクローンする．
```
cd /home4cluster
git clone https://github.com/karpathy/nanoGPT.git nanochat
```

## ステップ2:学習データの準備
LLMを学習するための学習データをダウンロードします．
LLM推論で用いたDockerコンテナを使用して，学習用データセットをダウンロードします．
以下のコマンドを実行してください．
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "cd /home4cluster/nanochat && pip install -q tiktoken && python data/shakespeare_char/prepare.py"
```

## ステップ3:1台で動作確認
node15の1台のみで動作確認を行います．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=1 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.0.15 \
      --master_port=29605 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=1 \
      --compile=False
  " 2>&1 | tee /home4cluster/logs/train/1node_$(date +%Y%m%d).log
```

## ステップ4:2台で動作確認
node15とnode16の2台で動作確認を行います．
`train.py`を修正するために以下のコマンドを実行します．
```
sed -i 's/use_fused = fused_available and device_type == .cuda./use_fused = False  # disabled: incompatible with GB10/' /home4cluster/nanochat/model.py
```
以下のコマンドをnode15で実行します．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=2 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.0.15 \
      --master_port=29604 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=2 \
      --compile=False
  " 2>&1 | tee /home4cluster/logs/train/2node_rj45_$(date +%Y%m%d).log
```
以下のコマンドをnode16で実行します．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=2 --nproc_per_node=1 \
      --node_rank=1 \
      --master_addr=10.0.0.15 \
      --master_port=29604 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=2 \
      --compile=False
  "
```

## ステップ5:4台で動作確認
node15~node18の4台で動作確認を行います．
以下のコマンドをnode15で実行してください．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=4 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.0.15 \
      --master_port=29606 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=4 \
      --compile=False
  " 2>&1 | tee /home4cluster/logs/train/4node_rj45_$(date +%Y%m%d).log
```
node16で実行します．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=4 --nproc_per_node=1 \
      --node_rank=1 \
      --master_addr=10.0.0.15 \
      --master_port=29606 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=4 \
      --compile=False
  "
```
node17で実行します．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=4 --nproc_per_node=1 \
      --node_rank=2 \
      --master_addr=10.0.0.15 \
      --master_port=29606 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=4 \
      --compile=False
  "
```
node18で実行します．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enP7s7 \
  -e GLOO_SOCKET_IFNAME=enP7s7 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=4 --nproc_per_node=1 \
      --node_rank=3 \
      --master_addr=10.0.0.15 \
      --master_port=29606 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=4 \
      --compile=False
  "
```


## ステップ6:2台で動作確認（QSFP）
node15で実行．
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  -e GLOO_SOCKET_IFNAME=enp1s0f0np0 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=2 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.1.1 \
      --master_port=29610 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=2 \
      --compile=False
  " 2>&1 | tee /home4cluster/logs/train/2node_qsfp_$(date +%Y%m%d).log
```
node16で実行
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_SOCKET_IFNAME=enp1s0f0np0 \
  -e GLOO_SOCKET_IFNAME=enp1s0f0np0 \
  -e NCCL_IB_DISABLE=1 \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=2 --nproc_per_node=1 \
      --node_rank=1 \
      --master_addr=10.0.1.1 \
      --master_port=29610 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 --log_interval=10 \
      --gradient_accumulation_steps=2 \
      --compile=False
  "
```



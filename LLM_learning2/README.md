# LLMの学習評価2

## ステップ1:nanochatの用意

```
cd /home4cluster
git clone https://github.com/karpathy/nanochat.git
```

```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "cd /home4cluster/nanochat && pip list | grep -E 'torch|triton'"
```

## ステップ2:必要モジュールのインストール
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    echo '=== インストール完了 ==='
```

```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    python -c '
import wandb
import pyarrow
import filelock
import jinja2
import tokenizers
import psutil
import requests
import rustbpe
import tiktoken
print(\"全モジュールのimport成功\")
'
  "
```

## ステップ3
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    cd /home4cluster/nanochat &&
    python -m scripts.tok_train
  "
```
```
docker run --rm \
  -v /home4cluster:/home4cluster \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    cd /home4cluster/nanochat &&
    python -m nanochat.dataset -n 10
  "
```

## ステップ4:1nodeで動作確認
```
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e NCCL_NET=Socket \
  -e NCCL_P2P_DISABLE=1 \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    cd /home4cluster/nanochat &&
    torchrun \
      --nnodes=1 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.0.15 \
      --master_port=29701 \
      -m scripts.base_train -- \
      --max-seq-len=2048 \
      --device-batch-size=21 \
      --total-batch-size=86016 \
      --window-pattern L
  " 2>&1 | tee /home4cluster/logs/train/nanochat_1node_windowL_$(date +%Y%m%d).log

docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
    cd /home4cluster/nanochat &&
    torchrun \
      --nnodes=1 --nproc_per_node=1 \
      --node_rank=0 \
      --master_addr=10.0.0.15 \
      --master_port=29701 \
      -m scripts.base_train -- \
      --max-seq-len=2048 \
      --device-batch-size=21 \
      --total-batch-size=86016 \
      --window-pattern L \
      --num-iterations=30
  " 2>&1 | tee /home4cluster/logs/train/nanochat_1node_speed_$(date +%Y%m%d).log
```


## ステップ5:Docker Imageを用意
```
# イメージのpull（約20GB，時間がかかります）
docker pull nvcr.io/nvidia/pytorch:25.09-py3
```
環境を確認．
```
docker run --rm --gpus all \
  nvcr.io/nvidia/pytorch:25.09-py3 \
  bash -c "python -c \"
import torch
print('PyTorch:', torch.__version__)
print('CUDA:', torch.version.cuda)
try:
    from flash_attn import flash_attn_func
    print('Flash Attention: available')
except ImportError:
    print('Flash Attention: not available')
\""
```

## ステップ6:1nodeで学習
まず，Dockerコンテナを起動します．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
Dockerコンテナを起動し，コンテナに入ったら，学習を実行します．
以下のコマンドを実行します．
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=1 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=43008 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_1node_$(date +%Y%m%d).log
```
2パターン目の学習
```
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=1 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=43008 \
  --window-pattern L \
  --num-iterations=100 \
2>&1 | tee /home4cluster/logs/train/nanochat_1node_$(date +%Y%m%d).log
```

## ステップ7:2nodeでの学習評価（QSFP）
QSFP接続した2nodeでの学習を行います．
まず，以下のコマンドをnode15とnode16で実行してDockerコンテナを起動してください．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
node15のコンテナ内で以下のコマンドを実行してください．
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=0 \
  --master_addr=10.0.1.1 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=86016 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_2node_qsfp_$(date +%Y%m%d).log
```
続いて，node16のコンテナ内で以下のコマンドを実行してください．
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=1 \
  --master_addr=10.0.1.1 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=86016 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_2node_qsfp_$(date +%Y%m%d).log
```


## ステップ8:2nodeでの学習評価（RJ45）
RJ45接続した2nodeでの学習を行います．
まず，以下のコマンドをnode15とnode16で実行してください．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
node15のコンテナ内で以下のコマンドを実行してください．
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=86016 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
node16のコンテナ内で以下のコマンドを実行してください．
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=1 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=86016 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```

## ステップ9:4nodeでの学習評価（RJ45）
全てのnodeでDockerコンテナを起動してください．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3

pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf
```
各nodeで順番に実行する．
```
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=172032 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=1 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=172032 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=2 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=172032 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=3 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=21 \
  --total-batch-size=172032 \
  --window-pattern L \
  --num-iterations=30 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```


## ステップ10:比較評価
### 1node
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
```
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=1 --node_rank=0 \
  --master_addr=10.0.0.18 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=2 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_1node_$(date +%Y%m%d).log
```

### 2node(QSFP)
```
# 全てのnodeで実行
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
```
# node15で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=0 \
  --master_addr=10.0.1.1 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=10 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_2node_qsfp_$(date +%Y%m%d).log
```
```
# node16で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=1 \
  --master_addr=10.0.1.1 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=10 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_2node_qsfp_$(date +%Y%m%d).log
```

### 2node(RJ45)
```
# 全てのnodeで実行
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
```
# mode15で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=10 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
# node16で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf

NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=2 --node_rank=1 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=10 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```

### node4(RJ45)
```
# 全てのnodeで実行
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -e WANDB_MODE=disabled \
  -e NANOCHAT_BASE_DIR=/home4cluster/nanochat_data \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/nanochat \
  nvcr.io/nvidia/pytorch:25.09-py3
```
```
# node15
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf


NCCL_IB_DISABLE=1
NCCL_NET=Socket 
NCCL_P2P_DISABLE=1 
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=0 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=5 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
# node16で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf


NCCL_IB_DISABLE=1
NCCL_NET=Socket 
NCCL_P2P_DISABLE=1 
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=1 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=5 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
# node17で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf


NCCL_IB_DISABLE=1
NCCL_NET=Socket 
NCCL_P2P_DISABLE=1
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=2 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=5 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```
```
# node18で実行
pip install -q wandb pyarrow filelock jinja2 tokenizers psutil requests rustbpe tiktoken &&
pip install -q --upgrade protobuf


NCCL_IB_DISABLE=1
NCCL_NET=Socket 
NCCL_P2P_DISABLE=1
NCCL_SOCKET_IFNAME=enP7s7 \
torchrun \
  --nproc_per_node=1 --nnodes=4 --node_rank=3 \
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=5 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_4node_rj45_$(date +%Y%m%d).log
```


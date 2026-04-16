# LLMの学習速度評価
以下のように，QSFP接続した状態を1つのnodeとして捉えて，LLMの推論を行う．
```
[node15+16] = 仮想GPU① (256GB)   ← 内部通信はQSFP
      ↕ RJ45 1Gbps
[node17+18] = 仮想GPU② (256GB)   ← 内部通信はQSFP
```

## 前準備
### ステップ1:動作確認
node15で以下のコマンドを順番に実行して，動作確認を行なってください．
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
  --master_addr=10.0.0.15 --master_port=29500 \
  -m scripts.base_train -- \
  --max-seq-len=2048 \
  --device-batch-size=2 \
  --total-batch-size=40960 \
  --window-pattern L \
  --num-iterations=500 \
2>&1 | tee /home4cluster/logs/train/nanochat_1node_$(date +%Y%m%d).log
```


### ステップ2:node15とnode16で　動作確認
2node（node15とnode16）での動作確認を行います．






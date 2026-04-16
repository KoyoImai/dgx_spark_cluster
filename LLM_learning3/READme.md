# LLMの学習速度評価
以下のように，QSFP接続した状態を1つのnodeとして捉えて，LLMの推論を行う．
```
[node15+16] = 仮想GPU① (256GB)   ← 内部通信はQSFP
      ↕ RJ45 1Gbps
[node17+18] = 仮想GPU② (256GB)   ← 内部通信はQSFP
```

## 前準備
### ステップ1:リポジトリのクローン
管理者node8で，リポジトリをクローンします．
```
cd /home4cluster
git clone https://github.com/llm-jp/Megatron-LM.git
cd Megatron-LM
```

### ステップ2:Dockerコンテナの起動
node15で以下を実行してください．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/Megatron-LM \
  nvcr.io/nvidia/pytorch:25.09-py3
```

### ステップ3:依存パッケージのインストール
```
cd /home4cluster/Megatron-LM
pip install -e .
```


### ステップ4：Shakespeareデータの準備
まず生テキストをダウンロードします．
コンテナ内で以下を実行してください．
```
mkdir -p /home4cluster/megatron_data
wget -O /home4cluster/megatron_data/shakespeare.txt \
  https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt
```
ダウンロードしたテキストをMegatron用のJSON形式に変換します．
```
python3 -c "
import json
with open('/home4cluster/megatron_data/shakespeare.txt', 'r') as f:
    text = f.read()
with open('/home4cluster/megatron_data/shakespeare.jsonl', 'w') as f:
    for line in text.split('\n\n'):
        line = line.strip()
        if line:
            json.dump({'text': line}, f)
            f.write('\n')
"
```
続いてMegatron用バイナリ形式に変換します．
```
wget -O /home4cluster/megatron_data/gpt2-vocab.json \
  https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-vocab.json
wget -O /home4cluster/megatron_data/gpt2-merges.txt \
  https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-merges.txt

python tools/preprocess_data.py \
  --input /home4cluster/megatron_data/shakespeare.jsonl \
  --output-prefix /home4cluster/megatron_data/shakespeare \
  --tokenizer-type GPT2BPETokenizer \
  --vocab-file /home4cluster/megatron_data/gpt2-vocab.json \
  --merge-file /home4cluster/megatron_data/gpt2-merges.txt \
  --append-eod \
  --workers 4
```


## 動作確認
### ステップ1：1nodeでの動作確認（node15）
node15のコンテナ内で以下を実行してください．
```
GPUS_PER_NODE=1
MASTER_ADDR=10.0.0.15
MASTER_PORT=29500
NNODES=1
NODE_RANK=0

torchrun \
  --nproc_per_node $GPUS_PER_NODE \
  --nnodes $NNODES \
  --node_rank $NODE_RANK \
  --master_addr $MASTER_ADDR \
  --master_port $MASTER_PORT \
  pretrain_gpt.py \
  --tensor-model-parallel-size 1 \
  --pipeline-model-parallel-size 1 \
  --num-layers 12 \
  --hidden-size 768 \
  --num-attention-heads 12 \
  --seq-length 1024 \
  --max-position-embeddings 1024 \
  --micro-batch-size 4 \
  --global-batch-size 16 \
  --train-iters 50 \
  --tokenizer-type GPT2BPETokenizer \
  --vocab-file /home4cluster/megatron_data/gpt2-vocab.json \
  --merge-file /home4cluster/megatron_data/gpt2-merges.txt \
  --data-path /home4cluster/megatron_data/shakespeare_text_document \
  --split 900,50,50 \
  --distributed-backend nccl \
  --lr 0.00015 \
  --min-lr 1.0e-5 \
  --lr-decay-style cosine \
  --weight-decay 1e-2 \
  --clip-grad 1.0 \
  --lr-warmup-iters 5 \
  --bf16 \
  --log-interval 10 \
  --log-throughput \
  2>&1 | tee /home4cluster/logs/train/megatron_1node_$(date +%Y%m%d).log
```

### ステップ2:2nodeでの動作確認
node15とnode16でそれぞれDockerコンテナを起動します．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/Megatron-LM \
  nvcr.io/nvidia/pytorch:25.09-py3
```
node15で以下のコマンドを実行してください．
```
NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node 1 \
  --nnodes 2 \
  --node_rank 0 \
  --master_addr 10.0.1.1 \
  --master_port 29500 \
  pretrain_gpt.py \
  --tensor-model-parallel-size 2 \
  --pipeline-model-parallel-size 1 \
  --num-layers 12 \
  --hidden-size 768 \
  --num-attention-heads 12 \
  --seq-length 1024 \
  --max-position-embeddings 1024 \
  --micro-batch-size 4 \
  --global-batch-size 16 \
  --train-iters 50 \
  --tokenizer-type GPT2BPETokenizer \
  --vocab-file /home4cluster/megatron_data/gpt2-vocab.json \
  --merge-file /home4cluster/megatron_data/gpt2-merges.txt \
  --data-path /home4cluster/megatron_data/shakespeare_text_document \
  --split 900,50,50 \
  --distributed-backend nccl \
  --lr 0.00015 \
  --min-lr 1.0e-5 \
  --lr-decay-style cosine \
  --weight-decay 1e-2 \
  --clip-grad 1.0 \
  --lr-warmup-iters 5 \
  --bf16 \
  --log-interval 10 \
  --log-throughput \
  2>&1 | tee /home4cluster/logs/train/megatron_2node_qsfp_$(date +%Y%m%d).log
```
node16で以下のコマンドを実行してください．
```
NCCL_SOCKET_IFNAME=enp1s0f0np0 \
torchrun \
  --nproc_per_node 1 \
  --nnodes 2 \
  --node_rank 1 \
  --master_addr 10.0.1.1 \
  --master_port 29500 \
  pretrain_gpt.py \
  --tensor-model-parallel-size 2 \
  --pipeline-model-parallel-size 1 \
  --num-layers 12 \
  --hidden-size 768 \
  --num-attention-heads 12 \
  --seq-length 1024 \
  --max-position-embeddings 1024 \
  --micro-batch-size 4 \
  --global-batch-size 16 \
  --train-iters 50 \
  --tokenizer-type GPT2BPETokenizer \
  --vocab-file /home4cluster/megatron_data/gpt2-vocab.json \
  --merge-file /home4cluster/megatron_data/gpt2-merges.txt \
  --data-path /home4cluster/megatron_data/shakespeare_text_document \
  --split 900,50,50 \
  --distributed-backend nccl \
  --lr 0.00015 \
  --min-lr 1.0e-5 \
  --lr-decay-style cosine \
  --weight-decay 1e-2 \
  --clip-grad 1.0 \
  --lr-warmup-iters 5 \
  --bf16 \
  --log-interval 10 \
  --log-throughput \
  2>&1 | tee /home4cluster/logs/train/megatron_2node_qsfp_$(date +%Y%m%d).log
```



### ステップ3:4node（TP=2×DP=2，QSFP+RJ45混在）
構成は以下のようになっています．
"""
rank 0 (node15) ─── TP グループ A ───  rank 1 (node16)
    ↕ DP通信 (RJ45)                         ↕ DP通信 (RJ45)
rank 2 (node17) ─── TP グループ B ───  rank 3 (node18)
"""
全node（node15〜18）でDockerコンテナを起動します．
```
docker run --gpus all -it --rm \
  --ipc=host --network=host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  --device=/dev/infiniband \
  -v /home4cluster:/home4cluster \
  -w /home4cluster/Megatron-LM \
  nvcr.io/nvidia/pytorch:25.09-py3
```



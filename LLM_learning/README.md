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
# node15 で実行
docker run --rm --gpus all \
  --network host --ipc host \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "
    cd /home4cluster/nanochat &&
    pip install -q tiktoken &&
    torchrun \
      --nnodes=1 \
      --nproc_per_node=1 \
      train.py config/train_shakespeare_char.py \
      --max_iters=100 \
      --log_interval=10
  " 2>&1 | tee /home4cluster/logs/train/1node_$(date +%Y%m%d).log
```

## ステップ4:2台で動作確認
node15とnode16の2台で動作確認を行います．





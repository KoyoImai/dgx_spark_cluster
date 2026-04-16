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


### ステップ2:node15とnode16で　動作確認
2node（node15とnode16）での動作確認を行います．






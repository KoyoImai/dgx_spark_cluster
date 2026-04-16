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





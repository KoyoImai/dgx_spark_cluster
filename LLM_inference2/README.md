# LLMの推論速度を比較2
LLM_inferenc edえは，lamma.cpp RPCを用いた場合において，lamma-benchで評価した．
ここでは，vLLM+Rayによる推論速度などの評価を行う．

**[参考1:vLLM](https://zenn.dev/karaage0703/articles/fcca40c614dffd)**
**[参考2:vLLM](https://licensecounter.jp/engineer-voice/blog/articles/20251219_dgx_spark_4vllm.html)**
**[参考3:vLLM](https://zenn.dev/t0d4/articles/7d8951b7c40fba)**

## 比較表
### 測定条件
- モデル：GPT-OSS-120B（MXFP4量子化）
- 推論エンジン：vLLM 0.11.0（nvcr.io/nvidia/vllm:25.11-py3）
- データセット：ShareGPT
- ベンチマークツール：vllm bench serve

### 結果一覧

| パターン | node数 | NW | num-prompts | Output tok/s | Peak Output tok/s | TTFT中央値 (ms) | TPOT中央値 (ms) |
|---|---:|---|---:|---:|---:|---:|---:|
| 1台 | 1 | - | 1 | 30.76 | 32.00 | 89.70 | 31.95 |
| 1台 | 1 | - | 10 | 67.98 | 115.00 | 1,328.91 | 80.59 |
| 1台 | 1 | - | 100 | 159.40 | 331.00 | 5,062.49 | 304.82 |
| 2台（QSFP） | 2 | QSFP 200Gbps | 1 | - | - | - | - |
| 2台（QSFP） | 2 | QSFP 200Gbps | 10 | - | - | - | - |
| 2台（QSFP） | 2 | QSFP 200Gbps | 100 | - | - | - | - |
| 2台（RJ45） | 2 | RJ45 1Gbps | 1 | - | - | - | - |
| 2台（RJ45） | 2 | RJ45 1Gbps | 10 | - | - | - | - |
| 2台（RJ45） | 2 | RJ45 1Gbps | 100 | - | - | - | - |
| 4台（RJ45） | 4 | RJ45 1Gbps | 1 | - | - | - | - |
| 4台（RJ45） | 4 | RJ45 1Gbps | 10 | - | - | - | - |
| 4台（RJ45） | 4 | RJ45 1Gbps | 100 | - | - | - | - |

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
for NUM_PROMPTS in 1 10 100; do
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
      --num-prompts ${NUM_PROMPTS} \
    2>&1 | tee -a /home4cluster/logs/vllm/1node_$(date +%Y%m%d).log
  sleep 5
done
```


# LLMの推論速度を比較2
LLM_inferenc edえは，lamma.cpp RPCを用いた場合において，lamma-benchで評価した．
ここでは，vLLM+Rayによる推論速度などの評価を行う．

**[参考1:vLLM](https://zenn.dev/karaage0703/articles/fcca40c614dffd)**
**[参考2:vLLM](https://licensecounter.jp/engineer-voice/blog/articles/20251219_dgx_spark_4vllm.html)**
**[参考3:vLLM](https://zenn.dev/t0d4/articles/7d8951b7c40fba)**

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

## ステップ3:
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

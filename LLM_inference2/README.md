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

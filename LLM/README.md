# DGX Spark4台を使用した言語モデルの利用
構築したDGX Spark4台の環境を使って大規模言語モデルを動かします。

## ステップ1：Singularity環境の用意
クラスタ上で大規模言語モデルを動かすための環境を用意します。
まず、管理者nodeで以下のコマンドを実行してください。
```
cd /home4cluster/
sudo mkdir containers
cd containers/
singularity pull docker://ghcr.io/ggml-org/llama.cpp:full-cuda
```


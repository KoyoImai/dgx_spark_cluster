# DGX Spark4台を使用した言語モデルの利用
構築したDGX Spark4台の環境を使って大規模言語モデルを動かします。

## ステップ1：Singularity環境の用意
クラスタ上で大規模言語モデルを動かすための環境を用意します。
まず、管理者nodeで以下のコマンドを実行してください。
```
cd /home4cluster/
sudo mkdir containers
cd containers/
sudo chmod 777 /home4cluster/containers
singularity pull docker://ghcr.io/ggml-org/llama.cpp:full-cuda
```
node15でSlurn jobとして実行して確認します。
```
srun --partition=pair1 --nodes=1 --gres=gpu:1 \
  singularity exec --nv /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi
```



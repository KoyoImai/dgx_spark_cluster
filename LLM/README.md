# DGX Spark4台を使用した言語モデルの利用
構築したDGX Spark4台の環境を使って大規模言語モデルを動かします。
使用するDGX Sparkの台数、QSFP、RJ45の差によって推論速度などにどのような影響が出るかを確かめます。
検証パターンをいかにまとめます。

| パターン | partition | node数 | NW | 備考 |
|---|---|---:|---|---|
| 1台 | pair1 | 1 | - | node15固定 |
| 2台（QSFP） | pair1 | 2 | QSFP 200Gbps | node15↔16 |
| 2台（QSFP） | pair2 | 2 | QSFP 200Gbps | node17↔18 |
| 2台（RJ45） | all | 2 | RJ45 1Gbps | node15↔16（同構成でNW変更） |
| 4台（RJ45） | all | 4 | RJ45 1Gbps | node15〜18 |
| 2台（RJ45） | all | 2 | RJ45 10Gbps | node15↔16（同構成でNW変更） |
| 4台（RJ45） | all | 4 | RJ45 10Gbps | node15〜18 |

## ステップ1：Singularity環境の用意
ここでは、クラスタ上で大規模言語モデルを動かすための環境を用意します。

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
また、4台のDGX Sparkで同時にGPUが使用可能かを確かめます。
以下のコマンドを実行して結果を確認してください。
```
mprg@spark-3894:/home4cluster/containers$ srun --partition=all --nodes=4 --gres=gpu:1 \
  singularity exec --nv \
  --env LD_LIBRARY_PATH=/app \
  /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi --query-gpu=name,memory.total,memory.free --format=csv
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
name, memory.total [MiB], memory.free [MiB]
NVIDIA GB10, [N/A], [N/A]
mprg@spark-3894:/home4cluster/containers$ srun --partition=all --nodes=4 --gres=gpu:1 \
  singularity exec --nv \
  --env LD_LIBRARY_PATH=/app \
  /home4cluster/containers/llama.cpp_full-cuda.sif \
  nvidia-smi
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
Sat Apr  4 09:08:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0  On |                  N/A |
| N/A   38C    P8              5W /  N/A  | Not Supported          |      1%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3821      G   /usr/lib/xorg/Xorg                      166MiB |
|    0   N/A  N/A            3963      G   /usr/bin/gnome-shell                    179MiB |
|    0   N/A  N/A            4862      G   .../7965/usr/lib/firefox/firefox        587MiB |
|    0   N/A  N/A           12599      G   /usr/bin/gnome-control-center            32MiB |
+-----------------------------------------------------------------------------------------+
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   36C    P8              4W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   35C    P8              3W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3338      G   /usr/lib/xorg/Xorg                      133MiB |
|    0   N/A  N/A            3491      G   /usr/bin/gnome-shell                     92MiB |
|    0   N/A  N/A            4415      G   .../7965/usr/lib/firefox/firefox        695MiB |
|    0   N/A  N/A            9715      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
|    0   N/A  N/A            3439      G   /usr/lib/xorg/Xorg                      111MiB |
|    0   N/A  N/A            3592      G   /usr/bin/gnome-shell                     90MiB |
|    0   N/A  N/A            4409      G   .../7965/usr/lib/firefox/firefox        490MiB |
|    0   N/A  N/A           10172      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
|   0  NVIDIA GB10                    On  |   0000000F:01:00.0 Off |                  N/A |
| N/A   35C    P8              4W /  N/A  | Not Supported          |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A            3644      G   /usr/lib/xorg/Xorg                      111MiB |
|    0   N/A  N/A            3787      G   /usr/bin/gnome-shell                     78MiB |
|    0   N/A  N/A            4333      G   ...exec/xdg-desktop-portal-gnome         76MiB |
|    0   N/A  N/A            4603      G   .../7965/usr/lib/firefox/firefox        479MiB |
|    0   N/A  N/A           11820      G   /usr/bin/gnome-control-center            37MiB |
+-----------------------------------------------------------------------------------------+
mprg@spark-3894:/home4cluster/containers$ 
```

## ステップ2：学習済み大規模言語モデルの用意
huggingfaceから学習済みパラメータをダウンロードしてきます。
以下のコマンドを実行してください。
```
cd /home4cluster/models

# ファイル1（49.9GB）
wget "https://huggingface.co/unsloth/Qwen3-235B-A22B-GGUF/resolve/main/Q2_K/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf"

# ファイル2（35.8GB）
wget "https://huggingface.co/unsloth/Qwen3-235B-A22B-GGUF/resolve/main/Q2_K/Qwen3-235B-A22B-Q2_K-00002-of-00002.gguf"
```


## ステップ3 : 大規模言語モデルで推論を実行


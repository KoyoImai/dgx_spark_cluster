# LLMの推論速度を比較

## RPC対応コンテナの作成
RPCに対応した仮想環境を作成します。
まず、以下のコマンドを作成してください。
```
# サンドボックス作成
sudo singularity build --sandbox \
  /tmp/llama-sandbox \
  /home4cluster/containers/llama.cpp_full-cuda.sif

# cmake インストール
sudo singularity exec \
  --writable --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  /tmp/llama-sandbox \
  bash -c "apt-get update && apt-get install -y cmake build-essential"

# llama.cpp クローン＆ビルド
sudo singularity exec \
  --writable \
  --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  /tmp/llama-sandbox \
  bash -c "
    export PATH=/usr/local/cuda/bin:\$PATH && \
    rm -rf /build && \
    mkdir -p /build && cd /build && \
    git clone https://github.com/ggml-org/llama.cpp.git . && \
    mkdir build && cd build && \
    cmake .. \
      -DGGML_CUDA=ON \
      -DGGML_RPC=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_CUDA_ARCHITECTURES=121 && \
    cmake --build . --parallel 20 && \
    cp /build/build/bin/* /app/
  "
```
作成したサンドボックスを`.sif`に変換します。
```
sudo singularity build \
  /home4cluster/containers/llama-rpc.sif \
  /tmp/llama-sandbox
```
RPCバイナリを確認します。
```
singularity exec /home4cluster/containers/llama-rpc.sif \
  ls /app/ | grep rpc
# 出力：rpc-server 
```
最後に1台推論で動作を確認してください。
```
singularity exec --nv \
  --bind /usr/local/cuda:/usr/local/cuda \
  --bind /home4cluster:/home4cluster \
  --env LD_LIBRARY_PATH=/app:/usr/local/cuda/lib64 \
  /home4cluster/containers/llama-rpc.sif \
  /app/llama-bench \
  --model /home4cluster/models/Qwen3-235B-A22B-Q2_K-00001-of-00002.gguf \
  --n-gpu-layers 99 \
  -p 128 -n 256 -r 3
```


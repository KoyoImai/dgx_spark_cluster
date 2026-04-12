# LLMの学習評価2

## ステップ1:nanochatの用意

```
cd /home4cluster
git clone https://github.com/karpathy/nanochat.git
```

## ステップ2

```
docker run --rm \
  -v /home4cluster:/home4cluster \
  nvcr.io/nvidia/vllm:25.11-py3 \
  bash -c "cd /home4cluster/nanochat && pip list | grep -E 'torch|triton'"
```


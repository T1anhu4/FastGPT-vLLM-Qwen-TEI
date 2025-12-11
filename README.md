<div align="right">
  <a href="README_EN.md">🌐 English Version</a>
</div>

# 🚀FastGPT-vLLM-Qwen-TEI全链路本地化部署

> 基于vLLM(Qwen2.5)的FastGPT高性能私有RAG知识库和Frp+Nginx公网访问的一站式部署方案。
> 专为国内内网环境优化，解决HuggingFace下载、Docker镜像加速及显存OOM等痛点。

![FastGPT](https://img.shields.io/badge/FastGPT-v4.x-blue)
![vLLM](https://img.shields.io/badge/vLLM-0.6.x-green)
![Qwen](https://img.shields.io/badge/Model-Qwen2.5--7B-violet)
![TEI](https://img.shields.io/badge/Embedding-BGE--M3-orange)

## 📖 项目简介

本项目提供了一套完整的FastGPT本地化部署实战指南，基于vLLM（推理加速）和TEI（向量加速）构建，充分发挥单卡显存性能，实现高并发、低延迟的知识库问答。

### 核心特性
- 🇨🇳 **国内网络优化**：提供HuggingFace镜像下载、Docker镜像代理配置全流程。
- ⚡ **高性能推理**：使用vLLM部署Qwen2.5-7B，使用TEI部署BGE-M3。
- 💾 **显存精细管理**：通过参数调优，实现LLM与Embedding模型在单卡显存下的稳定共存。
- 🌐 **公网访问**：集成FRP + Nginx方案，实现内网服务的公网安全访问。

## 🛠️ 技术栈
- **应用层**：FastGPT + MongoDB + PostgreSQL(pgvector)
- **推理层**：vLLM(Qwen2.5-7B-Instruct)
- **索引层**：Text Embeddings Inference(BAAI/bge-m3)
- **网络层**：FRP(内网穿透) + Nginx(反向代理)
- **容器化**：Docker & Docker Compose

## ⚡ 快速开始

### 1. 环境要求
- Ubuntu 20.04 / 22.04
- CUDA 12.1+
- 显存 ≥ 24GB (实际可以不用这么大，根据自己显存调节模型即可)

### 2. 国内加速模型下载

```bash
pip install huggingface_hub
export HF_ENDPOINT=https://hf-mirror.com

# Qwen2.5, 将'/your/path/'切换成你实际下载到本地的路径
huggingface-cli download --resume-download Qwen/Qwen2.5-7B-Instruct \
  --local-dir /your/path/Qwen2.5 

# BGE-M3, 将'/your/path/'切换成你实际下载到本地的路径
huggingface-cli download --resume-download BAAI/bge-m3 \
  --local-dir /your/path/bge-m3 \
  --exclude "*.DS_Store"
```

### 3.启动本地模型服务

### 3.1启动vLLM(对话模型服务)
- 注意：本项目为了保证单卡24G显存能够同时运行Embedding模型，我将vLLM的显存利用率严格限制在80%，实际可根据自己显卡的显存来调整这个比例。
```bash
# 将/your/path/Qwen2.5替换为您实际的模型下载路径
python -m vllm.entrypoints.openai.api_server \
    --model /your/path/Qwen2.5 \
    --served-model-name Qwen2.5 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.8 \
    --port 8000 \
    --host 0.0.0.0 \
    --api-key sk-123456
```
- --served-model-name Qwen2.5: 关键参数，强制指定模型名称，防止FastGPT报错404。
- --gpu-memory-utilization 0.8: 预留约20%显存给TEI模型使用。

### 3.2 启动TEI(向量索引服务)
- 注意：本项目使用Docker挂载本地模型目录，彻底规避容器内下载失败的问题。
```bash
# 将/your/path/bge-m3替换为您实际的模型下载路径
# 使用m.daocloud.io代理ghcr.io镜像
sudo docker run --gpus all -d --name tei-bge \
    -p 8008:80 \
    -v /your/path/bge-m3:/data \
    m.daocloud.io/ghcr.io/huggingface/text-embeddings-inference:latest \
    --model-id /data
```


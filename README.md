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
- 🇨🇳 **国内网络优化**：提供 HuggingFace 镜像下载、Docker 镜像代理配置全流程。
* ⚡ **高性能推理**：使用 vLLM 部署 Qwen2.5-7B，使用 TEI 部署 BGE-M3。
* 💾 **显存精细管理**：通过参数调优，实现 LLM与 Embedding 模型在单卡 24G 显存下的稳定共存。
* 🌐 **公网访问**：集成 FRP + Nginx 方案，实现内网服务的公网安全访问。

## 🛠️ 技术栈
- FastGPT + MongoDB + PostgreSQL
- vLLM(Qwen2.5-7B)
- TEI(BGE-M3)
- FRP + Nginx
- Docker & Docker Compose

## ⚡ 快速开始

### 1. 环境要求
- Ubuntu 20.04 / 22.04
- CUDA 12.1+
- 显存 ≥ 24GB (实际可以不用这么大，根据自己显存调节模型即可)

### 2. 国内加速模型下载

```bash
pip install huggingface_hub
export HF_ENDPOINT=https://hf-mirror.com

# Qwen2.5
huggingface-cli download --resume-download Qwen/Qwen2.5-7B-Instruct \
  --local-dir /your/path/Qwen2.5

# BGE-M3
huggingface-cli download --resume-download BAAI/bge-m3 \
  --local-dir /your/path/bge-m3 \
  --exclude "*.DS_Store"


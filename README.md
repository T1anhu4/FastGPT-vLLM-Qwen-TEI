# FastGPT-vLLM-Qwen-TEI
An end-to-end deployment solution for a high-performance private RAG knowledge base powered by FastGPT with vLLM (Qwen2.5), combined with public access via FRP and Nginx.
基于vLLM(Qwen2.5)的FastGPT高性能私有RAG知识库和Frp+Nginx公网访问的一站式部署方案。


# 🚀 FastGPT + vLLM 全链路本地化部署指南

> 基于 Ubuntu + 单卡 24G 显存的企业级知识库私有化方案。
> 专为**国内内网环境**优化，解决 HuggingFace 下载、Docker 镜像加速及显存 OOM 等痛点。

![FastGPT](https://img.shields.io/badge/FastGPT-v4.x-blue)
![vLLM](https://img.shields.io/badge/vLLM-0.6.x-green)
![Qwen](https://img.shields.io/badge/Model-Qwen2.5--7B-violet)
![TEI](https://img.shields.io/badge/Embedding-BGE--M3-orange)

## 📖 项目简介

本项目提供了一套完整的 FastGPT 本地化部署实战指南。不同于简单的 demo，本方案基于 **vLLM (推理加速)** 和 **TEI (向量加速)** 构建，能够充分榨干单张 24G 显卡（如 RTX 3090/4090）的性能，实现高并发、低延迟的知识库问答。

**核心特性：**
* 🇨🇳 **国内网络优化**：提供 HuggingFace 镜像下载、Docker 镜像代理配置全流程。
* ⚡ **高性能推理**：使用 vLLM 部署 Qwen2.5-7B，使用 TEI 部署 BGE-M3。
* 💾 **显存精细管理**：通过参数调优，实现 LLM与 Embedding 模型在单卡 24G 显存下的稳定共存。
* 🌐 **公网访问**：集成 FRP + Nginx 方案，实现内网服务的公网安全访问。

## 🛠️ 技术栈

* **应用层**：FastGPT + MongoDB + PostgreSQL (pgvector)
* **推理层**：vLLM (Qwen2.5-7B-Instruct)
* **索引层**：Text Embeddings Inference (BAAI/bge-m3)
* **网络层**：FRP (内网穿透) + Nginx (反向代理)
* **容器化**：Docker & Docker Compose

## ⚡ 快速开始

### 1. 环境准备
* 操作系统：Ubuntu 20.04/22.04
* 显卡驱动：CUDA 12.1+
* 显存要求：≥ 24GB (推荐 RTX 3090/4090 或 A10)

### 2. 模型下载 (国内加速方案)
本方案放弃 Docker 自动下载，采用更稳定的“宿主机下载+挂载”模式。

```bash
# 1. 安装工具
pip install huggingface_hub

# 2. 设置国内镜像
export HF_ENDPOINT=[https://hf-mirror.com](https://hf-mirror.com)

# 3. 下载 Qwen2.5 (对话模型)
huggingface-cli download --resume-download Qwen/Qwen2.5-7B-Instruct --local-dir /your/path/Qwen2.5

# 4. 下载 BGE-M3 (索引模型) - 关键：排除垃圾文件防止 403 错误
huggingface-cli download --resume-download BAAI/bge-m3 \
    --local-dir /your/path/bge-m3 \
    --exclude "*.DS_Store"

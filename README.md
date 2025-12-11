# FastGPT-vLLM-Qwen-TEI
An end-to-end deployment solution for a high-performance private RAG knowledge base powered by FastGPT with vLLM (Qwen2.5), combined with public access via FRP and Nginx.
基于vLLM(Qwen2.5)的FastGPT高性能私有RAG知识库和Frp+Nginx公网访问的一站式部署方案。


<div align="right">
  <a href="README_EN.md">🌐 English Version</a>
</div>

# 🚀 FastGPT + vLLM 全链路本地化部署指南

> 基于 Ubuntu + 单卡 24G 显存的企业级知识库私有化方案。
> 专为国内内网环境优化，解决 HuggingFace 下载、Docker 镜像加速及显存 OOM 等痛点。

![FastGPT](https://img.shields.io/badge/FastGPT-v4.x-blue)
![vLLM](https://img.shields.io/badge/vLLM-0.6.x-green)
![Qwen](https://img.shields.io/badge/Model-Qwen2.5--7B-violet)
![TEI](https://img.shields.io/badge/Embedding-BGE--M3-orange)

## 📖 项目简介

本项目提供了一套完整的 FastGPT 本地化部署实战指南，基于 vLLM（推理加速）和 TEI（向量加速）构建，充分发挥单卡 24G 显存性能，实现高并发、低延迟的知识库问答。

### 核心特性
- 🇨🇳 国内网络优化
- ⚡ 高性能推理
- 💾 显存精细管理
- 🌐 FRP + Nginx 公网访问方案

## 🛠️ 技术栈
- FastGPT + MongoDB + PostgreSQL
- vLLM (Qwen2.5-7B)
- TEI (BGE-M3)
- FRP + Nginx
- Docker & Docker Compose

## ⚡ 快速开始

### 1. 环境要求
- Ubuntu 20.04 / 22.04
- CUDA 12.1+
- 显存 ≥ 24GB

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


<div align="right">
  <a href="README.md">🇨🇳 中文版</a>
</div>

# 🚀 FastGPT + vLLM End-to-End Local Deployment Guide

> An enterprise-grade private RAG deployment solution based on Ubuntu + a single 24GB GPU.  
> Optimized for China mainland intranet environments, solving issues like HF model downloading, Docker image acceleration, and GPU OOM.

![FastGPT](https://img.shields.io/badge/FastGPT-v4.x-blue)
![vLLM](https://img.shields.io/badge/vLLM-0.6.x-green)
![Qwen](https://img.shields.io/badge/Model-Qwen2.5--7B-violet)
![TEI](https://img.shields.io/badge/Embedding-BGE--M3-orange)

## 📖 Overview

This project provides a complete practical guide for deploying FastGPT locally, using vLLM (inference acceleration) and TEI (embedding acceleration) to fully utilize a single 24GB GPU and achieve high concurrency and low latency.

### Key Features
- China-network optimization
- High-performance inference
- Fine-grained VRAM control
- Public access with FRP + Nginx

## 🛠️ Tech Stack
- FastGPT + MongoDB + PostgreSQL
- vLLM (Qwen2.5-7B)
- TEI (BGE-M3)
- FRP + Nginx
- Docker & Docker Compose

## ⚡ Quick Start

### 1. Requirements
- Ubuntu 20.04 / 22.04
- CUDA 12.1+
- VRAM ≥ 24GB

### 2. Model Download (China Accelerated)

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

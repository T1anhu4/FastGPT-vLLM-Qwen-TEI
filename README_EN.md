<div align="right">
<a href="README_EN.md"> 🌐  English Version</a>
</div>

#  🚀 FastGPT vLLM Qwen TEI Full Link Localization Deployment

>A one-stop deployment solution for FastGPT high-performance private RAG knowledge base and Frp+Nginx public network access based on vLLM (Qwen2.5)
>Specially optimized for domestic intranet environments, solving pain points such as HuggingFace downloads, Docker image acceleration, and out of memory (OOM) issues

![FastGPT]( https://img.shields.io/badge/FastGPT-blue )
![vLLM]( https://img.shields.io/badge/vLLM-0.12.0-green )
![Qwen]( https://img.shields.io/badge/Model-Qwen2.5--7B--Instruct-violet )
![TEI]( https://img.shields.io/badge/Embedding-BGE--M3-orange )

##  📖 Project Introduction

This project provides a complete set of practical guidelines for localized deployment of FastGPT, based on vLLM (inference acceleration) and TEI (vector acceleration), fully leveraging the performance of single card graphics memory to achieve high concurrency and low latency knowledge base Q&A

###Core Features
-🇨🇳 * * Domestic network optimization * *: Provides the full process of HuggingFace image download and Docker image proxy configuration
-  ⚡ ** High performance inference: deploying Qwen2.5-7B using vLLM and BGE-M3 using TEI
-  💾 ** Fine management of VRAM: Through parameter tuning, achieve stable coexistence of LLM and Embedding models in single card VRAM
-  🌐 ** Public Access * *: Integrated FRP+Nginx solution to achieve secure public access to internal network services

##  🛠️ tech stack
-Application layer: FastGPT+MongoDB+PostgreSQL (pgvector)
-* * Inference layer * *: vLLM (Qwen2.5-7B Instruction)
-* * Index Layer * *: Text Embeddings Inference (BAI/bge-m3)
-Network Layer: FRP (Internal Network Penetration)+Nginx (Reverse Proxy)
-Containerization: Docker&Docker Compose

##  ⚡ Quick Start

## 1. environment setup
The following is the machine configuration and environment that I personally deployed:
- Ubuntu 22.04
- CUDA 12.1
- RTX3090 24GB
- Python 3.10
- Docker 28.2.2
- Dcoker Compose 2.20.3

###1.1 Installing NVIDIA Drivers and CUDA
Ensure that Nvidia smi can output graphics card information normally. Recommend installing CUDA version 12.1 or above
Detailed installation tutorials can be searched for on your own, and this project will not go into too much detail

###1.2 Installing Docker and NVIDIA Container Toolkit
-Note: NVIDIA Container Toolkit must be installed, otherwise Docker containers cannot call the graphics card

```bash
# 1. Install Docker
curl -fsSL  https://get.docker.com  | bash -s docker --mirror Aliyun
systemctl enable --now docker

curl -L  https://github.com/docker/compose/releases/download/v2.20.3/docker-compose- `uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

#Verify installation
docker -v
docker-compose -v
#If it fails, you can ask AI to solve it~

# 2. Configure domestic image source (key: commas are strictly prohibited at the end of JSON arrays)
sudo tee /etc/docker/daemon.json <<EOF
{
"registry-mirrors": [
"[ https://docker.nju.edu.cn ]( https://docker.nju.edu.cn )",
"[ https://docker.m.daocloud.io ]( https://docker.m.daocloud.io )"
]
}
EOF

# 3. Install NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker

# 4. Restart Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

###1.3 Installing Python Environment
-Note: This project uses a virtual environment installed by Miniforge. Of course, you can choose your own local Python environment without using a virtual environment~
```bash
#Create and activate an environment
conda create -n llm python=3.10
conda activate llm

#Install dependencies
pip install vllm huggingface_hub
```

## 2. Model Download

```bash
pip install huggingface_hub
export HF_ENDPOINT= https://hf-mirror.com

# Qwen2.5, Switch '/your/path/' to the path you want to download
huggingface-cli download --resume-download Qwen/Qwen2.5-7B-Instruct \
--local-dir /your/path/Qwen2.5 

# BGE-M3, Switch '/your/path/' to the path you want to download
huggingface-cli download --resume-download BAAI/bge-m3 \
--local-dir /your/path/bge-m3 \
--exclude "*.DS_Store"
```

## 3. Start local model service
###3.1 Start vLLM (Qwen2.5)
-Note: In order to ensure that a single card with 24GB of VRAM can run the Embedding model simultaneously, I have strictly limited the VRAM utilization rate of vLLM to 80%. In practice, this ratio can be adjusted according to the VRAM of your own graphics card
-` -- served model name Qwen2.5 `: Key parameter, enforces the specification of model name to prevent FastGPT from reporting error 404
-GPU memory tilization 0.8: Reserve approximately 20% of video memory for use by TEI models
```bash
#Replace/yours/path/Qwen2.5 with the model path that was just downloaded
python -m vllm.entrypoints.openai.api_server \
--model /your/path/Qwen2.5 \
--served-model-name Qwen2.5 \
--max-model-len 8192 \
--gpu-memory-utilization 0.8 \
--port 8000 \
--host 0.0.0.0 \
--api-key sk-123456
```

###3.2 Start TEI (BGE-M3)
-Attention: This project uses Docker to mount the local model directory, completely avoiding the problem of download failure within the container
```bash
#Replace/yours/path/boge-m3 with the newly downloaded model path
#Use m.daocloud.io as a proxy for ghcr.io images
sudo docker run --gpus all -d --name tei-bge \
-p 8008:80 \
-v /your/path/bge-m3:/data \
m.daocloud.io/ghcr.io/huggingface/text-embeddings-inference:latest \
--model-id /data
```

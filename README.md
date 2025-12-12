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

## 1. 环境搭建
以下是我个人部署的机器配置和环境：
- Ubuntu 22.04
- CUDA 12.1
- RTX3090-24GB
- Python3.10
- Docker 28.2.2
- Dcoker Compose 2.20.3

### 1.1 安装NVIDIA驱动与CUDA
确保`nvidia-smi`可以正常输出显卡信息。推荐安装CUDA 12.1或以上版本。
详细的安装教程可以自行搜索，本项目不过多介绍。

### 1.2 安装Docker与NVIDIA Container Toolkit
- 注意：必须安装`NVIDIA Container Toolkit`，否则Docker容器无法调用显卡。

```bash
# 1. 安装Docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
systemctl enable --now docker

curl -L https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# 验证安装
docker -v
docker-compose -v
# 如失效，可以问AI解决下~

# 2. 配置国内镜像源(关键：JSON数组末尾严禁逗号)
sudo tee /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "[https://docker.nju.edu.cn](https://docker.nju.edu.cn)",
        "[https://docker.m.daocloud.io](https://docker.m.daocloud.io)"
    ]
}
EOF

# 3. 安装NVIDIA Container Toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker

# 4. 重启Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

###1.3 安装Python环境
- 注意：本项目使用的是`Miniforge`安装的虚拟环境，当然你也可以不用虚拟环境，用你自己的本地Python环境也行，自行选择~
```bash
# 创建并激活环境
conda create -n llm python=3.10
conda activate llm

# 安装依赖
pip install vllm huggingface_hub
```

## 2. 模型下载

```bash
pip install huggingface_hub
export HF_ENDPOINT=https://hf-mirror.com

# Qwen2.5, 将'/your/path/'切换成你想要下载的路径
huggingface-cli download --resume-download Qwen/Qwen2.5-7B-Instruct \
  --local-dir /your/path/Qwen2.5 

# BGE-M3, 将'/your/path/'切换成你想要下载的路径
huggingface-cli download --resume-download BAAI/bge-m3 \
  --local-dir /your/path/bge-m3 \
  --exclude "*.DS_Store"
```

## 3.启动本地模型服务
### 3.1启动vLLM(Qwen2.5)
- 注意：本项目为了保证单卡24G显存能够同时运行Embedding模型，我将vLLM的显存利用率严格限制在80%，实际可根据自己显卡的显存来调整这个比例。
- `--served-model-name Qwen2.5`: 关键参数，强制指定模型名称，防止FastGPT报错404。
- `--gpu-memory-utilization 0.8`: 预留约20%显存给TEI模型使用。
```bash
# 将/your/path/Qwen2.5替换为刚刚下载好的模型路径
python -m vllm.entrypoints.openai.api_server \
    --model /your/path/Qwen2.5 \
    --served-model-name Qwen2.5 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.8 \
    --port 8000 \
    --host 0.0.0.0 \
    --api-key sk-123456
```

### 3.2 启动TEI(BGE-M3)
- 注意：本项目使用Docker挂载本地模型目录，彻底规避容器内下载失败的问题。
```bash
# 将/your/path/bge-m3替换为刚刚下载好的模型路径
# 使用m.daocloud.io代理ghcr.io镜像
sudo docker run --gpus all -d --name tei-bge \
    -p 8008:80 \
    -v /your/path/bge-m3:/data \
    m.daocloud.io/ghcr.io/huggingface/text-embeddings-inference:latest \
    --model-id /data
```

## 4.部署FastGPT
### 4.1 创建目录与拉取配置文件
- 以下两个镜像自行选择下载,若无法下载可以在本仓库中手动下载。
- `docker-compose.yml`和`config.json`文件我已经上传在本仓库了。
```bash
# 国内镜像(阿里云)
bash <(curl -fsSL https://doc.fastgpt.cn/deploy/install.sh) --region=cn --vector=pg

# 非国内镜像(dockhub, ghcr)
bash <(curl -fsSL https://doc.fastgpt.cn/deploy/install.sh) --region=global --vector=pg
```

### 4.2 修改配置文件(可选)
- 通常情况下，`docker-compose.yml`无需修改。如果端口冲突，可修改`ports`映射。
- `config.json`用于系统级配置，建议保持默认，模型配置我们在Web界面进行。

### 4.3 启动FastGPT
```bash
# 启动所有服务(Mongo, Postgres, FastGPT, OneAPI)
docker-compose up -d
```
- 访问地址：`http://localhost:3000`(默认账号:`root`, 密码:`docker-compose.yml`里的`DEFAULT_ROOT_PSW`)
- 官方默认密码我记得是`1234`

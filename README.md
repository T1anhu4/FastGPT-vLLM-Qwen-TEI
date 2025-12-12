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

## 📖项目简介

本项目提供了一套完整的FastGPT本地化部署实战指南，基于vLLM（推理加速）和TEI（向量加速）构建，充分发挥单卡显存性能，实现高并发、低延迟的知识库问答。

### 核心特性
- 🇨🇳 **国内网络优化**：提供HuggingFace镜像下载、Docker镜像代理配置全流程。
- ⚡ **高性能推理**：使用vLLM部署Qwen2.5-7B，使用TEI部署BGE-M3。
- 💾 **显存精细管理**：通过参数调优，实现LLM与Embedding模型在单卡显存下的稳定共存。
- 🌐 **公网访问**：集成FRP + Nginx方案，实现内网服务的公网安全访问。

## 🛠️技术栈
- **应用层**：FastGPT + MongoDB + PostgreSQL(pgvector)
- **推理层**：vLLM(Qwen2.5-7B-Instruct)
- **索引层**：Text Embeddings Inference(BAAI/bge-m3)
- **网络层**：FRP(内网穿透) + Nginx(反向代理)
- **容器化**：Docker & Docker Compose

## ⚡快速开始

## 1. 环境搭建
以下是我个人部署的机器配置和环境：
- Ubuntu 22.04
- CUDA 12.1
- RTX3090 24GB
- Python 3.10
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

### 1.3 安装Python环境
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
- 本项目的修改如下：
`docker-compose.yml`
```bash
# 这个设置成自己服务器的公网IP就行，端口不变
S3_EXTERNAL_BASE_URL: http://123.456.789.112:7002

# fastgpt这里加了extra_hosts
...
fastgpt:
    container_name: fastgpt
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.14.3 # git
    ports:
     - 3000:3000
    networks:
      - fastgpt
    # 添加了extra_hosts
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      - mongo
      - sandbox
      - vectorDB
...

# fastgpt-mcp-server里也加了extra_hosts
...
fastgpt-mcp-server:
    container_name: fastgpt-mcp-server
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-mcp_server:v4.14.3
    networks:
     - fastgpt
    ports:
     - 3005:3000
    # 添加了extra_hosts
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: always
...
```

`config.json`
```bash
# 这个设置成自己服务器的公网IP就行，端口不变
...
  "mcpServerProxyEndpoint": "http://123.456.789.112:7003"
...
```

### 4.3 启动FastGPT
```bash
# 启动所有服务(Mongo, Postgres, FastGPT, OneAPI)
docker-compose up -d
```
- 访问地址：`http://localhost:3000`(默认账号:`root`, 密码:`docker-compose.yml`里的`DEFAULT_ROOT_PSW`)
- 官方默认密码我记得是`1234`

### 4.4 配置模型渠道
进入FastGPT -> 账号 -> 模型提供商 -> 模型渠道 -> 新建渠道：
1.Qwen2.5配置：
- 渠道名：想写啥写啥
- 协议类型：`OpenAI`
- 模型：`新增模型` -> `语言模型`
- 模型ID：`Qwen2.5`(这个也在vLLM启动命令里可以自行设置)
- 模型提供商：`通义千问`
- 别名：想写啥写啥
- 最大上下文：`12288`
- 知识库最大引用：`10000`
- 最大响应tokens：`2048`
- 右侧功能开关：✅工具调用 ✅知识库处理 ✅问题分类 ✅文本提取 ✅工具调用节点(可以按照自己的需求来设置)
- 设置完模型点击`确认`后选择`Qwen2.5`模型
- 代理地址：`http://172.17.0.1:8000/v1`(宿主机网关IP)
- API密钥：`sk-123456`(这个在使用vLLM启动模型时可自行设置，若启动命令没设置则此栏可以随便填)

2.BGE-M3配置：
- 渠道名：想写啥写啥
- 协议类型：`OpenAI`
- 模型：`新增模型` -> `索引模型`
- 模型ID：`BAAI/bge-m3`(这个也在vLLM启动命令里可以自行设置)
- 模型提供商：`OpenAI`
- 别名：想写啥写啥
- ✅ 归一化处理
- 并发请求数：`5`
- 默认分块长度：`512`
- 最大上下文：`8192`
- 设置完模型点击`确认`后选择`BAAI/bge-m3`模型
- 代理地址：`http://172.17.0.1:8000/v1`(宿主机网关IP)
- API密钥：`sk-aaabbb`(我这里启动时没设置，所以随便填了)

都配置完后可以分别对这两个模型进行`模型测试`下，测试下是否正常
正常的话可以进行下一步了，有异常的话用curl等工具检查下网络通信是否正常(Docker与宿主机的通信是否正常，通常都是防火墙拦截了)

## 5.公网访问(FRP + Nginx)
### 5.1 下载frps
- 注：云服务器和本地客户端都要下载解压这个包
```bash
# 创建个文件夹放frp文件
mkdir ./frp && cd ./frp

# 下载frp包
wget https://github.com/fatedier/frp/releases/download/v0.65.0/frp_0.65.0_linux_amd64.tar.gz

# 解压
tar -zxvf frp_0.65.0_linux_amd64.tar.gz
```

### 5.2 配置云服务器端(frps)
在当前`frp`目录下`nano ./frps.ini`进行配置:
```bash
[common]
# frps与frpc通信的端口(要和你frpc.ini的server_port一致)
bind_port = 7000
bind_addr = 0.0.0.0
# 日志路径
log_file = ./frps.log   
log_level =info
# 日志最大保存天数
log_max_days = 7        

# !!!强密码,这个token可以自己去随机生成一个,与frpc端必须一致
authentication_method = token
token = 8f7d6c5b4a3e2d1c9b8a7f6e5d4c3b2a

# 启用TLS - 必须
tls_enable = true
```
配置完成保存后就可以启动frps服务了
```bash
./frps -c ./frps.ini
```

### 5.3 配置本地客户端(frpc)
在当前`frp`目录下`nano ./frpc.ini`进行配置:
```bash
[common]
# 填写你的云服务器公网IP
server_addr = 此处填写云服务器公网IP
server_port = 7000
authentication_method = token
token = 8f7d6c5b4a3e2d1c9b8a7f6e5d4c3b2a

# 启用TLS - 必须
tls_enable = true

[fastgpt-app]
type = tcp
local_ip = 127.0.0.1
local_port = 3000
remote_port = 7001

[fastgpt-mcp]
type = tcp
local_ip = 127.0.0.1
local_port = 3005
remote_port = 7003

[fastgpt-s3]
type = tcp
local_ip = 127.0.0.1
local_port = 9000
remote_port = 7002
```
配置完成保存后就可以启动frps服务了
```bash
./frpc -c ./frpc.ini
```

### 5.4 安全组和防火墙开放
云服务器的安全组得开放以下几个端口:
`80` `7000` `7001` `7002` `7003`

云服务器和本地客户端都得开放以上几个端口：
```bash
sudo ufw allow 80/tcp
sudo ufw allow 7000/tcp
sudo ufw allow 7001/tcp
sudo ufw allow 7002/tcp
sudo ufw allow 7003/tcp

sudo ufw reload
```

### 5.5 配置Nginx
在云服务器上新建个配置文件
```sudo nano /etc/nginx/conf.d/fastgpt.conf```

直接粘贴，然后对三处进行修改地址：
```bash
# 1. 主应用, 根据自己服务器的实际公网IP进行修改
server {
    listen 80;
    server_name 此处修改为你的服务器公网IP + 端口(例如：123.456.789.112:7001);

    location / {
        proxy_pass http://127.0.0.1:7001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # WebSocket支持(FastGPT打字机效果需要)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# 2. S3文件服务, 根据自己服务器的实际公网IP进行修改
server {
    listen 80;
    server_name 此处修改为你的服务器公网IP + 端口(例如：123.456.789.112:7002);

    # 允许上传大文件,这个大小自行设置
    client_max_body_size 1000m;

    location / {
        proxy_pass http://127.0.0.1:7002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# 3. MCP 服务, 根据自己服务器的实际公网IP进行修改
server {
    listen 80;
    server_name 此处修改为你的服务器公网IP + 端口(例如：123.456.789.112:7003);

    location / {
        proxy_pass http://127.0.0.1:7003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

保存好后重启Nginx服务：
```bash
sudo nginx -t
sudo systemctl reload nginx
```
🤝贡献
欢迎提交Issue

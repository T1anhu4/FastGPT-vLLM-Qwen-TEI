<div align="right">
<a href="README.md"> 🌐  简体中文</a>
</div>

# 🚀FastGPT vLLM Qwen TEI Full Link Localization Deployment

> A one-stop deployment solution for FastGPT high-performance private RAG knowledge base and Frp+Nginx public network access based on vLLM (Qwen2.5).
> Specially optimized for domestic intranet environments, out of memory (OOM) issues.

![FastGPT]( https://img.shields.io/badge/FastGPT-blue )
![vLLM]( https://img.shields.io/badge/vLLM-0.12.0-green )
![Qwen]( https://img.shields.io/badge/Model-Qwen2.5--7B--Instruct-violet )
![TEI]( https://img.shields.io/badge/Embedding-BGE--M3-orange )

## 📖Project Introduction

This project provides a complete set of practical guidelines for localized deployment of FastGPT, based on vLLM (inference acceleration) and TEI (vector acceleration), fully leveraging the performance of single card graphics memory to achieve high concurrency and low latency knowledge base Q&A.

### Core Features
- ⚡ High performance inference: deploying Qwen2.5-7B using vLLM and BGE-M3 using TEI.
- 💾 Fine management of VRAM: Through parameter tuning, achieve stable coexistence of LLM and Embedding models in single card VRAM.
- 🌐 Public Access: Integrated FRP+Nginx solution to achieve secure public access to internal network services.

## 🛠️tech stack
- Application layer: FastGPT+MongoDB+PostgreSQL(pgvector)
- Inference layer: vLLM (Qwen2.5-7B Instruction)
- Index Layer: Text Embeddings Inference (BAI/bge-m3)
- Network Layer: FRP(Internal Network Penetration)+Nginx(Reverse Proxy)
- Containerization: Docker&Docker Compose

## ⚡Quick Start

## 1. Environment setup
The following is the machine configuration and environment that I personally deployed:
- Ubuntu 22.04
- CUDA 12.1
- RTX3090 24GB
- Python 3.10
- Docker 28.2.2
- Dcoker Compose 2.20.3

### 1.1 Installing NVIDIA Drivers and CUDA
Ensure that `nvidia-smi` can output graphics card information normally. Recommend installing CUDA version 12.1 or above.
Detailed installation tutorials can be searched for on your own, and this project will not go into too much detail.

### 1.2 Installing Docker and NVIDIA Container Toolkit
- Note: `NVIDIA Container Toolkit` must be installed, otherwise Docker containers cannot call the graphics card.

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

### 1.3 Installing Python Environment
- Note: This project uses a virtual environment installed by Miniforge. Of course, you can choose your own local Python environment without using a virtual environment~
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
### 3.1 Start vLLM (Qwen2.5)
- Note: In order to ensure that a single card with 24GB of VRAM can run the Embedding model simultaneously, I have strictly limited the VRAM utilization rate of vLLM to 80%. In practice, this ratio can be adjusted according to the VRAM of your own graphics card.
- `--served-model-name Qwen2.5`: Key parameter, enforces the specification of model name to prevent FastGPT from reporting error 404.
- `--gpu-memory-utilization 0.8`: Reserve approximately 20% of video memory for use by TEI models.
```bash
# Replace/yours/path/Qwen2.5 with the model path that was just downloaded
python -m vllm.entrypoints.openai.api_server \
--model /your/path/Qwen2.5 \
--served-model-name Qwen2.5 \
--max-model-len 8192 \
--gpu-memory-utilization 0.8 \
--port 8000 \
--host 0.0.0.0 \
--api-key sk-123456
```

### 3.2 Start TEI (BGE-M3)
- Attention: This project uses Docker to mount the local model directory, completely avoiding the problem of download failure within the container.
```bash
# Replace/yours/path/boge-m3 with the newly downloaded model path
# Use m.daocloud.io as a proxy for ghcr.io images
sudo docker run --gpus all -d --name tei-bge \
-p 8008:80 \
-v /your/path/bge-m3:/data \
m.daocloud.io/ghcr.io/huggingface/text-embeddings-inference:latest \
--model-id /data
```
## 4. Deploy FastGPT
### 4.1 Creating directories and pulling configuration files
- Please choose to download the following two images yourself. If you are unable to download them, you can manually download them from this repository.
- I have uploaded the files `docker-compose.yml` and `config.json` to this repository.
```bash
# Domestic mirror (Alibaba Cloud)
bash <(curl -fsSL  https://doc.fastgpt.cn/deploy/install.sh ) --region=cn --vector=pg

# Non domestic images (dockhub, ghcr)
bash <(curl -fsSL  https://doc.fastgpt.cn/deploy/install.sh ) --region=global --vector=pg
```

###4.2 Modify configuration file (optional)
- Normally, `docker-compose.yml` does not require modification. If there is a port conflict, the `ports` mapping can be modified.
- `config. json` is used for system level configuration, it is recommended to keep it as default. We configure the model through the web interface.
- The modifications to this project are as follows:
`docker-compose.yml`
```bash
# Just set this as the public IP of your own server, keep the port unchanged
S3_EXTERNAL_BASE_URL:  http://123.456.789.112:7002

# Fastgpt has added extra_ hosts here
...
fastgpt:
container_name: fastgpt
image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.14.3 # git
ports:
- 3000:3000
networks:
- fastgpt
# Added extra-hosts
extra_hosts:
- "host.docker.internal:host-gateway"
depends_on:
- mongo
- sandbox
- vectorDB
...

# The fastgpt mcp server has also added extra-hosts
...
fastgpt-mcp-server:
container_name: fastgpt-mcp-server
image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-mcp_server:v4.14.3
networks:
- fastgpt
ports:
- 3005:3000
# Added extra-hosts
extra_hosts:
- "host.docker.internal:host-gateway"
restart: always
...
```

`config.json`
```bash
# Just set this as the public IP of your own server, keep the port unchanged
...
"mcpServerProxyEndpoint": " http://123.456.789.112:7003 "
...
```

### 4.3 Starting FastGPT
```bash
# Start all services (Mongo, Postgres, FastGPT, OneAPI)
docker-compose up -d
```
- Access address: `http://localhost:3000 `(Default account: 'root', password: 'Docker-compose. yml' in 'DEFAULT_SOOT_PSW').
- The official default password, I remember, is `1234`.

### 4.4 Configuring Model Channels
Go to `FastGPT` -> `Account` -> `Model Provider` -> `Model Channel` -> `New Channel`:

1. Qwen2.5 configuration:
- Channel name: Write whatever you want
- Protocol Type: `OpenAI`
- Model: `New Model` -> `Language Model'`
- Model ID: `Qwen2.5` (which can also be set in the vLLM startup command.)
- Model provider: `Tongyi Qianwen`
- Alias: Write whatever you want
- Maximum context: `12288`
- Maximum citation in the knowledge base: `10000`
- Maximum response tokens: `2048`
- Right function switch: ✅ Tool invocation ✅ Knowledge base processing ✅ Problem classification ✅ text extraction ✅ Tool call node (can be set according to one's own needs)
- After setting up the model, click `confirm` and select `Qwen2.5` model
- Proxy address: `http://172.17.0.1:8000/v1 `(Host gateway IP)
- API key: `sk-123456` (This can be set by yourself when using vLLM to start the model. If the startup command is not set, this field can be filled in freely.)

2. BGE-M3 configuration:
- Channel name: Write whatever you want
- Protocol Type: `OpenAI`
- Model: `New Model` -> `Index Model`
- Model ID: `BAAI/bge-m3` (this can also be set in the vLLM startup command.)
- Model provider: `OpenAI`
- Alias: Write whatever you want
- ✅  normalization
- Number of concurrent requests: `5`
- Default block length: `512`
- Maximum context: `8192`
- After setting up the model, click 'confirm' and select 'BAII/bge-m3' model
- Proxy address:`http://172.17.0.1:8000/v1 `(Host gateway IP)
- API key: `sk-aaabbb` (I didn't set it when I started it here, so I filled it in randomly.)

After configuring both models, you can perform 'model testing' separately to check if they are functioning properly.
If everything is normal, you can proceed to the next step. If there are any abnormalities, use tools such as curl to check if the network communication is normal (Docker's communication with the host machine is usually blocked by the firewall).

## 5. Public network access (FRP+Nginx)
### 5.1 Download FRP
- Note: Both the cloud server and local client need to download and decompress this package.
```bash
# Create a folder to store FRP files
mkdir ./frp && cd ./frp

# Download FRP package
wget  https://github.com/fatedier/frp/releases/download/v0.65.0/frp_0.65.0_linux_amd64.tar.gz

# Relieve stress
tar -zxvf frp_0.65.0_linux_amd64.tar.gz
```

### 5.2 Configuring the cloud server side (frps)
Configure nano./frps. ini in the current frp directory:
```bash
[common]
# The communication port between frps and frpc should be consistent with the server port in your frpc.ini file
bind_port = 7000
bind_addr = 0.0.0.0
# Log path
log_file = ./frps.log   
log_level =info
# Maximum storage days for logs
log_max_days = 7        

# !!!Strong password, this token can be randomly generated by oneself and must be consistent with the frpc end
authentication_method = token
token = 8f7d6c5b4a3e2d1c9b8a7f6e5d4c3b2a

# Enable TLS - Required
tls_enable = true
```

After the configuration is completed and saved, the frps service can be started.

```bash
./frps -c ./frps.ini
```

### 5.3 Configuring Local Client (frpc)
Configure nano./frpc. ini in the current frp directory:
```bash
[common]
# Fill in your cloud server's public IP address
Serverless=Fill in the public IP address of the cloud server here
server_port = 7000
authentication_method = token
token = 8f7d6c5b4a3e2d1c9b8a7f6e5d4c3b2a

# Enable TLS - Required
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

After the configuration is completed and saved, the frps service can be started.

```bash
./frpc -c ./frpc.ini
```

### 5.4 Security Group and Firewall Openness
The security group of the cloud server needs to open the following ports:

`80` `7000` `7001` `7002` `7003`

Both cloud servers and local clients must open the above ports:
```bash
sudo ufw allow 80/tcp
sudo ufw allow 7000/tcp
sudo ufw allow 7001/tcp
sudo ufw allow 7002/tcp
sudo ufw allow 7003/tcp

sudo ufw reload
```

### 5.5 Configuring Nginx
Create a new configuration file on the cloud server
```sudo nano /etc/nginx/conf.d/fastgpt.conf```

Paste directly and then modify the address in three places:
```bash
# 1. Main application, modify according to the actual public IP address of its own server
server {
listen 80;
Change the server name here to your server's public IP and port (e.g. 123.456.789.112:7001);

location / {
proxy_pass  http://127.0.0.1:7001 ;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#WebSocket support (FastGPT typewriter effect required)
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
}
}

# 2. S3 file service, modify according to the actual public IP address of your own server
server {
listen 80;
Change the server name here to your server's public IP and port (e.g. 123.456.789.112:7002);

# Allow uploading large files, set the size by yourself
client_max_body_size 1000m;

location / {
proxy_pass  http://127.0.0.1:7002 ;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
}

# 3. MCP service, modify according to the actual public IP of your server
server {
    listen 80;
    server_name should be modified to your server's public IP + port (e.g., 123.456.789.112:7003);

    location / {
        proxy_pass http://127.0.0.1:7003;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

After saving, restart the Nginx service:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

At this time, accessing the public network at `123.456.789.112:7001` will allow you to enter the fastgpt page

# 🤝 Contribution
### Welcome to submit an issue

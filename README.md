# FastGPT-vLLM-Qwen-TEI
An end-to-end deployment solution for a high-performance private RAG knowledge base powered by FastGPT with vLLM (Qwen2.5), combined with public access via FRP and Nginx.
基于vLLM(Qwen2.5)的FastGPT高性能私有RAG知识库和Frp+Nginx公网访问的一站式部署方案。


# FastGPT 企业级本地知识库全链路部署指南 (完整版 v2.0)

> **文档日期：** 2025-12-11
> **环境架构：** Ubuntu 22.04 + NVIDIA GPU (24G VRAM)
> **核心组件：** FastGPT + vLLM (Qwen2.5-7B) + TEI (BGE-M3) + Docker + FRP + Nginx
> **文档目标：** 记录从零开始在纯本地算力下搭建可公网访问的企业级 AI 知识库的全过程，重点解决国内网络环境下的资源获取与配置问题。

---

## 📖 第一章：网络环境与部署策略

在部署 AI 模型时，网络环境决定了 90% 的难度。针对**外网**和**国内内网**，部署策略截然不同。

### 1.1 场景 A：国际互联网络 (Global Access)
* **适用场景：** 海外服务器，或拥有通畅的国际互联网访问权限。
* **部署策略：** 直接运行 Docker，容器内部自动下载模型。
* **命令示例：**
    ```bash
    docker run -d --gpus all -p 8008:80 ghcr.io/huggingface/text-embeddings-inference --model-id BAAI/bge-m3
    ```

### 1.2 场景 B：中国局域网/受限网络 (China Intranet) —— **[本次实战环境]**
* **适用场景：** 国内服务器，无法直连 HuggingFace，GitHub 访问慢或中断，Docker Hub 被墙。
* **部署策略：**
    1.  **镜像加速：** 必须配置 Docker 镜像代理（如南京大学源）。
    2.  **模型代理：** 必须设置 `HF_ENDPOINT=https://hf-mirror.com`。
    3.  **“外科手术式”部署：** 放弃容器自动下载，改为 **宿主机手动下载 + 挂载目录** 启动（彻底规避容器内网络不稳定导致的 EOF 错误）。
    4.  **垃圾文件规避：** 下载时必须排除 `.DS_Store` 等导致 403 错误的文件。

---

## 🛠️ 第二章：基础环境修复

### 2.1 Docker 镜像加速配置 (解决启动失败)
国内环境必须配置镜像加速，但 `daemon.json` 格式极其严格。

* **文件路径：** `/etc/docker/daemon.json`
* **⚠️ 避坑指南：** JSON 数组最后一项**绝对不能**有逗号，否则 Docker 无法启动 (`unable to configure daemon`)。
* **正确内容：**
    ```json
    {
      "registry-mirrors": [
        "[https://docker.nju.edu.cn](https://docker.nju.edu.cn)",
        "[https://docker.m.daocloud.io](https://docker.m.daocloud.io)"
      ]
    }
    ```
* **重启命令：**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

### 2.2 修复 iptables 缺失 (解决网络创建失败)
Ubuntu 重启后，Docker 可能报错 `iptables failed: no such file or directory`。
* **修复命令：**
    ```bash
    sudo ln -s /usr/sbin/iptables-legacy /usr/sbin/iptables
    ```

---

## 📥 第三章：模型下载 (解决网络阻断)

### 3.1 安装下载工具
```bash
conda activate llm
pip install huggingface_hub

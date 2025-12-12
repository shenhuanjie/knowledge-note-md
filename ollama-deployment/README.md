# Ollama Docker Compose 部署指南

本指南介绍如何使用 Docker Compose 部署 Ollama 服务，并将数据挂载到当前目录。

## 目录结构

```
ollama-deployment/
├── docker-compose.yml    # Docker Compose 配置文件
├── ollama_data/          # Ollama 数据目录（自动创建）
└── README.md             # 本说明文件
```

## 部署步骤

### 1. 进入部署目录

```bash
cd ollama-deployment
```

### 2. 启动服务

```bash
# 后台启动服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

### 3. 使用 Ollama

#### 3.1 拉取模型

```bash
# 进入容器内部
 docker exec -it ollama bash

# 拉取模型
ollama pull llama3
ollama pull mistral:7b
ollama pull gemma:2b
```

#### 3.2 运行模型

```bash
# 交互式对话
ollama run llama3

# 单次生成
ollama run llama3 "你好，Ollama！"
```

#### 3.3 使用 API

```bash
# 生成响应
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "你好，Ollama！"}'

# 对话
curl http://localhost:11434/api/chat -d '{"model": "llama3", "messages": [{"role": "user", "content": "你好，Ollama！"}]}'
```

#### 3.4 使用 Web UI

在浏览器中访问：http://localhost:3000

## 管理服务

```bash
# 停止服务
docker-compose down

# 停止并删除数据卷（谨慎使用）
docker-compose down -v

# 更新镜像
docker-compose pull
docker-compose up -d

# 重启服务
docker-compose restart
```

## 数据管理

- Ollama 数据（模型、配置等）存储在 `./ollama_data` 目录中
- 备份数据只需复制 `ollama_data` 目录即可
- 恢复数据只需将备份的 `ollama_data` 目录放回原位置

## 端口说明

| 端口 | 用途 |
|------|------|
| 11434 | Ollama API 端口 |
| 3000 | Ollama Web UI 端口 |

## 资源限制

默认配置限制了 Ollama 服务的资源使用：
- CPU：限制 4 核，保留 2 核
- 内存：限制 8GB，保留 4GB

可以根据实际硬件情况调整 `docker-compose.yml` 中的 `deploy.resources` 部分。

## 常见问题

### 1. 服务无法启动

查看日志以获取详细错误信息：
```bash
docker-compose logs ollama
```

### 2. 端口被占用

修改 `docker-compose.yml` 中的端口映射，例如：
```yaml
ports:
  - "11435:11434"  # 将主机端口改为11435
```

### 3. 模型拉取缓慢

配置 Docker 镜像加速，修改 `/etc/docker/daemon.json` 文件：
```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn", "https://hub-mirror.c.163.com"]
}
```

然后重启 Docker 服务：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

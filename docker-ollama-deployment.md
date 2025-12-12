# 使用 Docker 部署 Ollama 教程

## 1. 环境准备

### 1.1 安装 Docker

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
# 重新登录或执行以下命令使权限生效
su - $USER
```

#### CentOS/RHEL
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
# 重新登录或执行以下命令使权限生效
su - $USER
```

#### macOS
下载并安装 [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/)

#### Windows
下载并安装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)

### 1.2 验证 Docker 安装

```bash
docker --version
docker info
docker run hello-world
```

## 2. 部署 Ollama

### 2.1 拉取 Ollama 镜像

```bash
docker pull ollama/ollama:latest
```

### 2.2 运行 Ollama 容器

```bash
docker run -d \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  --restart always \
  ollama/ollama:latest
```

参数说明：
- `-d`: 后台运行容器
- `-v ollama:/root/.ollama`: 挂载卷，用于持久化 Ollama 模型和配置
- `-p 11434:11434`: 映射端口，使外部可以访问 Ollama API
- `--name ollama`: 为容器指定名称
- `--restart always`: 容器退出时自动重启

### 2.3 查看容器状态

```bash
docker ps
docker logs -f ollama
```

## 3. 使用 Ollama

### 3.1 进入容器内部

```bash
docker exec -it ollama bash
```

### 3.2 拉取模型

在容器内部执行：

```bash
ollama pull llama3
# 或拉取其他模型
ollama pull gemma:7b
ollama pull mistral
```

### 3.3 运行模型

在容器内部执行：

```bash
ollama run llama3
# 退出对话使用 /bye
```

### 3.4 外部访问 Ollama API

可以使用 curl 或其他 HTTP 客户端访问 Ollama API：

```bash
# 拉取模型
curl http://localhost:11434/api/pull -d '{"name": "llama3"}'

# 生成响应
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "你好，Ollama！"}'

# 对话
curl http://localhost:11434/api/chat -d '{"model": "llama3", "messages": [{"role": "user", "content": "你好，Ollama！"}]}}'
```

## 4. 安装 Ollama Web UI（可选）

### 4.1 拉取并运行 Web UI 容器

```bash
docker run -d \
  -p 3000:8080 \
  --name ollama-webui \
  --restart always \
  -e OLLAMA_API_BASE_URL=http://ollama:11434/api \
  --link ollama \
  ghcr.io/ollama-webui/ollama-webui:main
```

### 4.2 访问 Web UI

在浏览器中访问：http://localhost:3000

## 5. 常见问题处理

### 5.1 容器无法启动

```bash
# 查看容器日志
docker logs ollama

# 检查端口是否被占用
lsof -i :11434
netstat -tuln | grep 11434
```

### 5.2 模型拉取缓慢

可以使用国内镜像加速：

```bash
# 修改 Docker 配置，添加镜像加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn", "https://hub-mirror.c.163.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 5.3 外部无法访问 Ollama API

- 确保 Docker 容器的端口映射正确
- 检查防火墙设置，确保 11434 端口开放
- 对于云服务器，检查安全组规则

## 6. 管理 Ollama 容器

```bash
# 停止容器
docker stop ollama

# 启动容器
docker start ollama

# 重启容器
docker restart ollama

# 删除容器
docker stop ollama
docker rm ollama

# 更新镜像
docker pull ollama/ollama:latest
docker stop ollama
docker rm ollama
docker run -d ... ollama/ollama:latest
```

## 7. 总结

通过 Docker 部署 Ollama 是一种简单、高效的方式，可以快速搭建本地大语言模型服务。本教程介绍了从环境准备到容器部署、模型使用的完整流程，以及常见问题的处理方法。

使用 Docker 部署的优点：
- 隔离性好，不会影响系统其他组件
- 部署简单，一键启动
- 易于管理和更新
- 可移植性强，支持多种操作系统

希望本教程对你有所帮助！

# Ollama Docker Compose 部署指南

本指南介绍如何使用 Docker Compose 部署 Ollama 服务，并将数据挂载到当前目录。

## 部署状态

✅ **服务已成功部署并运行**

### 访问方式
- **API 地址**：http://localhost:11434
- **Web UI 地址**：http://localhost:3000
- **数据存储**：`./ollama_data` 目录（已自动创建）

### 容器状态
| 容器名称 | 镜像 | 状态 | 端口映射 |
|---------|------|------|---------|
| ollama | ollama/ollama:latest | ✅ Running | 0.0.0.0:11434->11434/tcp |
| ollama-webui | ghcr.io/ollama-webui/ollama-webui:main | ✅ Running | 0.0.0.0:3000->8080/tcp |

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

### 3. 验证服务

```bash
# 检查 API 状态
curl http://localhost:11434/api/tags
# 预期输出：{"models":[]}（表示服务正常但尚未拉取模型）
```

## 使用 Ollama

### 1. 拉取模型

```bash
# 进入容器内部
 docker exec -it ollama bash

# 拉取模型示例
ollama pull llama3           # 拉取 Llama 3 模型
ollama pull mistral:7b       # 拉取 Mistral 7B 模型
ollama pull gemma:2b         # 拉取 Gemma 2B 模型
ollama pull qwen:7b          # 拉取 Qwen 7B 模型（中文支持较好）

# 退出容器
exit
```

### 2. 运行模型

#### 2.1 交互式对话

```bash
# 启动交互式对话
 docker exec -it ollama ollama run llama3

# 对话示例
>>> 你好，Ollama！
>>> 写一首关于春天的诗
>>> /bye  # 退出对话
```

#### 2.2 单次生成

```bash
# 直接生成响应
docker exec -it ollama ollama run llama3 "你好，Ollama！"

# 将输出保存到文件
docker exec -it ollama ollama run llama3 "写一首关于秋天的诗" > autumn-poem.txt
```

### 3. 使用 API

#### 3.1 生成响应

```bash
# 基本生成
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "你好，Ollama！"}'

# 带参数的生成
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "写一篇关于人工智能的短文", "temperature": 0.7, "num_predict": 512}'
```

#### 3.2 对话模式

```bash
curl http://localhost:11434/api/chat -d '{"model": "llama3", "messages": [{"role": "user", "content": "你好，Ollama！"}, {"role": "assistant", "content": "你好！我是 Ollama，很高兴为你服务。"}, {"role": "user", "content": "什么是机器学习？"}]}'
```

#### 3.3 拉取模型（API 方式）

```bash
curl http://localhost:11434/api/pull -d '{"name": "llama3"}'
```

### 4. 使用 Web UI

在浏览器中访问：http://localhost:3000

Web UI 功能包括：
- 模型管理（拉取、删除、查看）
- 交互式对话
- 提示词模板管理
- 对话历史记录
- 参数调整
- API 密钥管理

## 服务管理

### 1. 基本管理命令

```bash
# 停止服务
docker-compose down

# 停止并删除数据卷（谨慎使用，会丢失所有模型数据）
docker-compose down -v

# 重启服务
docker-compose restart

# 更新镜像并重启服务
docker-compose pull
docker-compose up -d
```

### 2. 查看日志

```bash
# 查看所有服务日志
docker-compose logs -f

# 查看特定服务日志
docker-compose logs -f ollama
docker-compose logs -f ollama-webui
```

### 3. 进入容器

```bash
# 进入 Ollama 容器
docker exec -it ollama bash

# 进入 Web UI 容器
docker exec -it ollama-webui bash
```

## 数据管理

### 1. 数据备份

```bash
# 备份数据到压缩文件
tar -czf ollama_backup_$(date +%Y%m%d_%H%M%S).tar.gz ./ollama_data
```

### 2. 数据恢复

```bash
# 停止服务
docker-compose down

# 恢复数据
rm -rf ./ollama_data
tar -xzf ollama_backup_20251212_100000.tar.gz

# 重启服务
docker-compose up -d
```

## 配置说明

### 1. 端口配置

默认端口映射：
| 端口 | 用途 |
|------|------|
| 11434 | Ollama API 端口 |
| 3000 | Ollama Web UI 端口 |

如需修改端口，编辑 `docker-compose.yml` 文件中的 `ports` 部分：

```yaml
ports:
  - "自定义端口:11434"  # 修改 API 端口
```

### 2. 资源限制

默认资源配置：
- CPU：限制 4 核，保留 2 核
- 内存：限制 8GB，保留 4GB

如需调整，编辑 `docker-compose.yml` 文件中的 `deploy.resources` 部分：

```yaml
deploy:
  resources:
    limits:
      cpus: "4"
      memory: "8g"
    reservations:
      cpus: "2"
      memory: "4g"
```

## 常见问题

### 1. 服务无法启动

查看日志以获取详细错误信息：
```bash
docker-compose logs ollama
```

### 2. 端口被占用

```bash
# 检查端口占用情况
lsof -i :11434
netstat -tuln | grep 11434

# 修改端口映射后重启服务
docker-compose down
docker-compose up -d
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
docker-compose restart
```

### 4. 内存不足

- 选择更小的模型（如 2B、7B 模型）
- 调整 `docker-compose.yml` 中的内存限制
- 关闭其他占用资源的应用

## 最佳实践

1. **选择合适的模型**：根据硬件资源选择合适大小的模型
2. **定期备份数据**：定期备份 `ollama_data` 目录，防止数据丢失
3. **更新镜像**：定期更新 Ollama 镜像以获取最新功能和安全修复
4. **配置资源限制**：根据硬件情况合理配置资源限制，避免资源耗尽
5. **使用 Web UI**：对于不熟悉命令行的用户，推荐使用 Web UI 进行操作

## 相关链接

- [Ollama 官方文档](https://ollama.com/docs)
- [Ollama Web UI GitHub](https://github.com/ollama-webui/ollama-webui)
- [Docker Compose 官方文档](https://docs.docker.com/compose/)

## 版本信息

- **部署时间**：2025-12-12
- **Ollama 镜像**：ollama/ollama:latest
- **Web UI 镜像**：ghcr.io/ollama-webui/ollama-webui:main
- **Docker Compose 版本**：3.8+

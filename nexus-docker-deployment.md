# Nexus 部署与 Docker 镜像管理指南

## 1. Nexus 简介

Nexus Repository Manager 是一个强大的仓库管理工具，支持多种格式的软件包管理，包括 Docker 镜像、Maven 构件、npm 包等。使用 Nexus 作为 Docker 镜像仓库可以实现：

- 私有镜像的集中管理
- 镜像的版本控制和访问控制
- 减少外部依赖，提高部署速度
- 支持镜像缓存，节省带宽

## 2. 部署 Nexus

### 2.1 使用 Docker 部署 Nexus

#### 2.1.1 安装 Docker

确保系统已安装 Docker：

```bash
# 检查 Docker 版本
docker --version
```

如果未安装 Docker，请参考 [Docker 官方文档](https://docs.docker.com/get-docker/) 进行安装。

#### 2.1.2 创建 Nexus 数据目录

```bash
# 创建数据目录并设置权限
sudo mkdir -p /opt/nexus/data
sudo chown -R 200 /opt/nexus/data
sudo chmod -R 775 /opt/nexus/data
```

#### 2.1.3 启动 Nexus 容器

```bash
docker run -d \
  --name nexus \
  --restart always \
  -p 8081:8081 \
  -p 8082:8082 \
  -p 8083:8083 \
  -v /opt/nexus/data:/nexus-data \
  sonatype/nexus3:latest
```

**端口说明**：
- 8081：Nexus Web 界面端口
- 8082：Docker Hosted 仓库端口（用于推送镜像）
- 8083：Docker Group 或 Proxy 仓库端口（用于拉取镜像）

#### 2.1.4 验证 Nexus 启动

```bash
# 查看容器状态
docker ps

# 查看 Nexus 日志
docker logs -f nexus
```

当看到类似 `Started Sonatype Nexus OSS` 的日志时，Nexus 已成功启动。

## 3. 初始配置 Nexus

### 3.1 访问 Nexus Web 界面

在浏览器中访问：`http://服务器IP:8081`

### 3.2 登录 Nexus

1. 点击右上角的 "Sign In" 按钮
2. 初始用户名：`admin`
3. 初始密码位于容器内的 `/nexus-data/admin.password` 文件中：

   ```bash
   docker exec -it nexus cat /nexus-data/admin.password
   ```

4. 首次登录后，系统会要求修改密码并配置匿名访问

### 3.3 创建 Docker 仓库

#### 3.3.1 创建 Docker Hosted 仓库

用于推送本地构建的 Docker 镜像：

1. 登录 Nexus Web 界面
2. 点击左侧菜单栏的 "Server administration and configuration"（齿轮图标）
3. 点击 "Repositories"，然后点击 "Create repository"
4. 选择 "docker (hosted)" 类型
5. 配置仓库：
   - **Name**: `docker-hosted`
   - **HTTP**: 勾选 "Enable Docker V1 API"（可选，兼容旧版本 Docker）
   - **HTTP Port**: `8082`
   - **Deployment policy**: 选择 `Allow redeploy`
   - **Storage**: 保持默认设置
6. 点击 "Create repository" 完成创建

#### 3.3.2 创建 Docker Proxy 仓库

用于缓存外部 Docker Hub 镜像：

1. 点击 "Create repository"
2. 选择 "docker (proxy)" 类型
3. 配置仓库：
   - **Name**: `docker-proxy`
   - **Remote storage**: `https://registry-1.docker.io`
   - **Docker Index**: 选择 "Use Docker Hub"
   - **HTTP Port**: `8083`
4. 点击 "Create repository" 完成创建

#### 3.3.3 创建 Docker Group 仓库

用于统一访问 Hosted 和 Proxy 仓库：

1. 点击 "Create repository"
2. 选择 "docker (group)" 类型
3. 配置仓库：
   - **Name**: `docker-group`
   - **HTTP Port**: `8084`
   - **Group**: 将 `docker-hosted` 和 `docker-proxy` 添加到 "Members"
4. 点击 "Create repository" 完成创建

## 4. 配置 Docker 客户端

### 4.1 配置 Docker 信任 Nexus 仓库

#### 4.1.1 配置 insecure-registries（HTTP 方式）

编辑 Docker 配置文件 `/etc/docker/daemon.json`：

```json
{
  "insecure-registries": ["服务器IP:8082", "服务器IP:8083", "服务器IP:8084"]
}
```

重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 4.1.2 配置 HTTPS（生产环境推荐）

对于生产环境，建议配置 SSL 证书，使用 HTTPS 访问 Nexus 仓库。

### 4.2 登录 Nexus Docker 仓库

使用 `docker login` 命令登录到 Nexus Docker 仓库：

```bash
docker login 服务器IP:8082
```

输入 Nexus 的用户名和密码（admin/修改后的密码）。

## 5. 推送 Docker 镜像到 Nexus

### 5.1 标记本地镜像

```bash
# 标记本地镜像，格式：仓库地址/仓库名/镜像名:标签
docker tag nginx:latest 服务器IP:8082/docker-hosted/nginx:latest
```

### 5.2 推送镜像到 Nexus

```bash
docker push 服务器IP:8082/docker-hosted/nginx:latest
```

### 5.3 验证镜像推送

在 Nexus Web 界面中，点击左侧菜单栏的 "Browse"，选择 `docker-hosted` 仓库，查看推送的镜像。

## 6. 从 Nexus 拉取 Docker 镜像

### 6.1 从 Hosted 仓库拉取镜像

```bash
docker pull 服务器IP:8082/docker-hosted/nginx:latest
```

### 6.2 从 Group 仓库拉取镜像

Group 仓库可以同时访问 Hosted 和 Proxy 仓库：

```bash
# 拉取本地推送的镜像
docker pull 服务器IP:8084/docker-group/nginx:latest

# 拉取外部镜像（会自动缓存到 Nexus）
docker pull 服务器IP:8084/docker-group/ubuntu:latest
```

### 6.3 验证镜像拉取

使用 `docker images` 命令查看拉取的镜像：

```bash
docker images
```

## 7. Nexus Docker 镜像管理最佳实践

### 7.1 合理规划仓库结构

- **Hosted 仓库**：用于存储内部构建的镜像
- **Proxy 仓库**：用于缓存外部镜像
- **Group 仓库**：提供统一的访问入口

### 7.2 设置访问控制

1. 在 Nexus Web 界面中，点击左侧菜单栏的 "Security" → "Roles"
2. 创建自定义角色，设置仓库的访问权限
3. 点击 "Security" → "Users"，创建用户并分配角色

### 7.3 配置镜像清理策略

1. 点击左侧菜单栏的 "Server administration and configuration" → "Tasks"
2. 点击 "Create task"
3. 选择 "Purge unused Docker manifests and images"
4. 配置清理规则，例如：
   - **Repository**: 选择需要清理的仓库
   - **Remove images older than**: 设置保留时间
   - **Remove images when unused for**: 设置未使用时间
5. 配置任务执行频率
6. 点击 "Create task" 完成创建

### 7.4 监控 Nexus 性能

- 查看 Nexus 系统状态：`http://服务器IP:8081/#admin/system/status`
- 监控资源使用情况：`http://服务器IP:8081/#admin/system/metrics`

## 8. 常见问题处理

### 8.1 推送镜像时出现 "401 Unauthorized"

- 确保已使用正确的用户名和密码登录
- 检查用户是否有仓库的写权限

### 8.2 拉取镜像时出现 "404 Not Found"

- 检查镜像名称和标签是否正确
- 检查仓库配置是否正确
- 对于 Proxy 仓库，检查远程仓库地址是否可达

### 8.3 镜像推送速度慢

- 检查网络连接
- 对于 Proxy 仓库，检查远程仓库的访问速度

### 8.4 Nexus 内存不足

- 调整 Nexus 容器的内存限制：
  ```bash
  docker run -d \
    --name nexus \
    --restart always \
    -p 8081:8081 \
    -p 8082:8082 \
    -p 8083:8083 \
    -v /opt/nexus/data:/nexus-data \
    -e JAVA_OPTS="-Xms2g -Xmx4g" \
    sonatype/nexus3:latest
  ```

## 9. 总结

通过本指南，您已学会如何：

1. 使用 Docker 部署 Nexus Repository Manager
2. 配置 Nexus 初始设置
3. 创建 Docker 仓库（Hosted、Proxy、Group）
4. 配置 Docker 客户端访问 Nexus
5. 推送本地镜像到 Nexus
6. 从 Nexus 拉取镜像
7. 配置 Nexus 最佳实践和常见问题处理

Nexus 作为强大的仓库管理工具，可以有效管理 Docker 镜像，提高团队的开发和部署效率。合理配置和使用 Nexus，可以实现镜像的集中管理、版本控制和安全访问，为企业级容器化应用提供可靠的支持。

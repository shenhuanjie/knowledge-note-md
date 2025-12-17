# Nexus + Gitea + Gitea Actions 构建 CI/CD 工作流

## 1. 环境准备

### 1.1 组件介绍
- **Nexus**: 开源的仓库管理系统，用于存储 Maven、Docker 等 artifacts，支持私有仓库和代理仓库
- **Gitea**: 轻量级的 Git 服务，支持 Git 托管、Gitea Actions CI/CD、Issue 管理等功能
- **Gitea Actions**: Gitea 内置的 CI/CD 功能，兼容 GitHub Actions 语法，支持自托管 Runner

### 1.2 系统要求
- 操作系统: Linux (推荐 Ubuntu 22.04 LTS)
- CPU: 至少 4 核（推荐 8 核）
- 内存: 至少 8GB（推荐 16GB）
- 磁盘: 至少 50GB 可用空间（推荐 100GB SSD）
- 网络: 稳定的网络连接

### 1.3 前置条件

#### 1.3.1 安装 Docker
```bash
# 更新包索引
sudo apt update

# 安装依赖包
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# 添加 Docker 官方 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 添加 Docker 源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# 将当前用户添加到 docker 组（避免使用 sudo）
sudo usermod -aG docker $USER

# 重启 Docker 服务
sudo systemctl restart docker
```

#### 1.3.2 安装 Docker Compose
```bash
# 下载 Docker Compose 二进制文件
sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.6/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 添加执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose --version
```

#### 1.3.3 配置防火墙
```bash
# 允许 SSH 访问
sudo ufw allow 22

# 允许 Nexus 端口
sudo ufw allow 8081/tcp
sudo ufw allow 8082/tcp
sudo ufw allow 8083/tcp
sudo ufw allow 8084/tcp

# 允许 Gitea 端口
sudo ufw allow 3000/tcp
sudo ufw allow 2222/tcp

# 启用防火墙
sudo ufw enable

# 查看防火墙状态
sudo ufw status
```

## 2. Nexus 配置

### 2.1 部署 Nexus

创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'
services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: always
    ports:
      - "8081:8081"
      - "8082:8082"  # Docker Hosted Repository
      - "8083:8083"  # Docker Proxy Repository
    volumes:
      - nexus_data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx4g -XX:MaxDirectMemorySize=2g

volumes:
  nexus_data:
    driver: local
```

启动 Nexus：
```bash
docker-compose up -d
```

### 2.2 初始化 Nexus

#### 2.2.1 启动 Nexus
```bash
# 创建 Nexus 数据目录
sudo mkdir -p /opt/nexus/data
sudo chown -R 200:200 /opt/nexus/data

# 创建 docker-compose.yml 文件
cat > docker-compose.yml << EOF
version: '3.8'
services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: always
    ports:
      - "8081:8081"
      - "8082:8082"  # Docker Hosted Repository
      - "8083:8083"  # Docker Proxy Repository
      - "8084:8084"  # Docker Group Repository
    volumes:
      - /opt/nexus/data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx4g -XX:MaxDirectMemorySize=2g
EOF

# 启动 Nexus
docker-compose up -d

# 查看 Nexus 启动状态
docker logs -f nexus
```

#### 2.2.2 首次登录配置
1. **访问 Nexus Web UI**：在浏览器中输入 `http://<nexus-ip>:8081`
2. **获取初始密码**：
   ```bash
   docker exec -it nexus cat /nexus-data/admin.password
   ```
3. **登录**：使用用户名 `admin` 和获取到的初始密码登录
4. **修改密码**：按照提示设置新的管理员密码
5. **完成向导**：选择 "Enable anonymous access"（可选），点击 "Finish"

### 2.3 配置仓库

#### 2.3.1 创建 Maven 仓库

1. **创建 Maven Releases 仓库**：
   - 点击左侧菜单 **Settings**（齿轮图标）→ **Repositories** → **Create repository**
   - 选择 **maven2 (hosted)** 类型
   - 填写配置：
     - **Name**: `maven-releases`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **Version policy**: `Release`
     - **Layout policy**: `Strict`
     - **Deployment policy**: `Allow redeploy`
   - 点击 **Create repository**

2. **创建 Maven Snapshots 仓库**：
   - 点击 **Create repository** → 选择 **maven2 (hosted)**
   - 填写配置：
     - **Name**: `maven-snapshots`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **Version policy**: `Snapshot`
     - **Layout policy**: `Strict`
     - **Deployment policy**: `Allow redeploy`
   - 点击 **Create repository**

3. **创建 Maven Proxy 仓库**（代理 Maven Central）：
   - 点击 **Create repository** → 选择 **maven2 (proxy)**
   - 填写配置：
     - **Name**: `maven-central`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **Remote storage**: `https://repo1.maven.org/maven2/`
     - **Layout policy**: `Strict`
     - **Proxy**: 
       - HTTP proxy: 可选，根据网络环境配置
   - 点击 **Create repository**

4. **创建 Maven Group 仓库**（聚合多个仓库）：
   - 点击 **Create repository** → 选择 **maven2 (group)**
   - 填写配置：
     - **Name**: `maven-public`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **Group**: 
       - Member repositories: 从左侧添加 `maven-releases`、`maven-snapshots`、`maven-central`
     - **Layout policy**: `Strict`
   - 点击 **Create repository**

#### 2.3.2 创建 Docker 仓库

1. **创建 Docker Hosted 仓库**：
   - 点击 **Create repository** → 选择 **docker (hosted)**
   - 填写配置：
     - **Name**: `docker-hosted`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **HTTP**: 勾选并设置端口 `8082`
     - **V1 API Support**: 可选，根据需要勾选
     - **Deployment policy**: `Allow redeploy`
   - 点击 **Create repository**

2. **创建 Docker Proxy 仓库**（代理 Docker Hub）：
   - 点击 **Create repository** → 选择 **docker (proxy)**
   - 填写配置：
     - **Name**: `docker-proxy`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **HTTP**: 勾选并设置端口 `8083`
     - **Remote storage**: `https://registry-1.docker.io`
     - **Index type**: `Docker Hub`
   - 点击 **Create repository**

3. **创建 Docker Group 仓库**（聚合多个 Docker 仓库）：
   - 点击 **Create repository** → 选择 **docker (group)**
   - 填写配置：
     - **Name**: `docker-group`
     - **Online**: 勾选
     - **Storage**: 
       - Blob store: `default`
     - **HTTP**: 勾选并设置端口 `8084`
     - **Group**: 
       - Member repositories: 从左侧添加 `docker-hosted`、`docker-proxy`
   - 点击 **Create repository**

#### 2.3.3 配置用户和权限

1. **创建部署用户**：
   - 点击左侧菜单 **Security** → **Users** → **Create local user**
   - 填写配置：
     - **User ID**: `deployment`
     - **First name**: `Deployment`
     - **Last name**: `User`
     - **Email**: `deployment@example.com`
     - **Password**: 设置强密码
     - **Status**: `Active`
     - **Roles**: 选择 `nx-deployment`
   - 点击 **Create local user**

2. **验证仓库访问**：
   - Maven 仓库 URL：`http://<nexus-ip>:8081/repository/maven-public/`
   - Docker 仓库 URL：`http://<nexus-ip>:8084`

## 3. Gitea 配置

### 3.1 部署 Gitea

#### 3.1.1 准备部署环境
```bash
# 创建 Gitea 数据目录
sudo mkdir -p /opt/gitea/data
sudo chown -R 1000:1000 /opt/gitea/data

# 创建 docker-compose.yml 文件
cat > docker-compose.yml << EOF
version: '3.8'
services:
  gitea:
    image: gitea/gitea:1.21.11
    container_name: gitea
    restart: always
    ports:
      - "3000:3000"      # HTTP 端口
      - "2222:22"        # SSH 端口
    volumes:
      - /opt/gitea/data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      # 基本设置
      - GITEA__server__DOMAIN=<your-domain-or-ip>
      - GITEA__server__ROOT_URL=http://<your-domain-or-ip>:3000/
      - GITEA__server__HTTP_PORT=3000
      - GITEA__server__SSH_PORT=2222
      
      # 数据库设置（默认 SQLite，如需 MySQL/PostgreSQL 请修改）
      - GITEA__database__DB_TYPE=sqlite3
      
      # Actions 设置
      - GITEA__actions__ENABLED=true
      
      # 日志设置
      - GITEA__log__LEVEL=info
      - GITEA__log__MODE=console
    depends_on:
      - git
  
  git:
    image: alpine/git:latest
    container_name: git
    restart: always
    volumes:
      - /opt/gitea/data:/data
EOF

# 启动 Gitea
docker-compose up -d

# 查看 Gitea 启动状态
docker logs -f gitea
```

### 3.2 初始化 Gitea

#### 3.2.1 首次访问配置
1. **访问 Gitea Web UI**：在浏览器中输入 `http://<gitea-ip>:3000`
2. **填写初始配置**：
   - **数据库设置**：
     - 数据库类型：选择 `SQLite3`（简单部署推荐）或 `MySQL`/`PostgreSQL`（生产环境推荐）
   - **应用基本设置**：
     - 站点标题：输入站点名称，例如 `My Gitea Server`
     - 仓库根目录：保持默认 `/data/git/repositories`
     - Git LFS 根目录：保持默认 `/data/git/lfs`
     - 运行用户：保持默认 `git`
     - SSH 服务器端口：`2222`
     - HTTP 端口：`3000`
     - Gitea 基本 URL：`http://<gitea-ip>:3000/`
   - **可选设置**：
     - 启用邮件服务（可选）
     - 启用 Captcha（可选）
   - **管理员账号设置**：
     - 用户名：设置管理员用户名，例如 `admin`
     - 密码：设置强密码
     - 确认密码：再次输入密码
     - 邮箱：输入管理员邮箱
3. **完成安装**：点击 **立即安装**

#### 3.2.2 验证安装
- 尝试登录 Gitea
- 检查是否可以正常访问首页

### 3.3 创建用户和仓库

#### 3.3.1 创建普通用户
1. **登录管理员账号**
2. 点击顶部导航栏 **管理** → **用户管理** → **创建用户**
3. 填写用户信息：
   - 用户名：例如 `developer`
   - 密码：设置密码
   - 确认密码：再次输入密码
   - 邮箱：输入用户邮箱
   - 全名：可选
   - 状态：选择 `激活`
4. 点击 **创建用户**

#### 3.3.2 创建测试仓库
1. **登录 Gitea**（可以使用管理员账号或普通用户）
2. 点击右上角 **+** 号 → **新建仓库**
3. 填写仓库信息：
   - 仓库名称：例如 `demo-project`
   - 仓库描述：可选
   - 可见性：选择 `公开` 或 `私有`
   - 初始化仓库：
     - 勾选 **使用 README.md 初始化仓库**
     - 可选：添加 `.gitignore` 和 `LICENSE`
4. 点击 **创建仓库**

## 4. Gitea Actions 配置

### 4.1 启用 Actions 功能

1. **登录管理员账号**
2. 点击顶部导航栏 **管理** → **站点管理**
3. 点击左侧菜单 **设置** → **功能开关**
4. 确保 **Actions** 选项已勾选
5. 点击 **保存设置**

### 4.2 配置自托管 Runner

#### 4.2.1 在 Gitea 中创建 Runner
1. **进入仓库**：点击要配置 Runner 的仓库
2. 点击左侧菜单 **Actions** → **Runners**
3. 点击 **创建自托管 Runner**
4. 选择 **Linux** 作为 Runner 操作系统
5. 复制页面上显示的注册命令（包含 `./config.sh` 命令）

#### 4.2.2 在 Runner 服务器上安装和配置

1. **准备 Runner 服务器**：
   - 使用与 Gitea 服务器相同的 Linux 系统
   - 确保已安装 Docker 和 Docker Compose
   - 确保可以访问 Gitea 服务器

2. **下载 Runner 二进制文件**：
   ```bash
   # 创建 Runner 目录
   mkdir -p /opt/gitea-runner
   cd /opt/gitea-runner
   
   # 下载 Runner（根据 Gitea 版本选择正确的 Runner 版本）
   GITEA_VERSION=$(curl -s https://api.github.com/repos/go-gitea/gitea/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//')
   RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//')
   
   # 下载 Runner
   curl -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -o actions-runner.tar.gz
   
   # 解压 Runner
   tar xzf actions-runner.tar.gz
   
   # 添加执行权限
   chmod +x ./run.sh ./config.sh
   ```

3. **注册 Runner**：
   ```bash
   # 使用之前复制的注册命令，例如：
   ./config.sh --url http://<gitea-ip>:3000/<owner>/<repo> --token <registration-token>
   
   # 按照提示输入：
   # - Runner name: 自定义 Runner 名称，例如 `my-runner`
   # - Runner group: 保持默认 `Default`
   # - Labels: 可选，添加自定义标签，例如 `linux,x64,docker`
   # - Work folder: 保持默认 `_work`
   ```

4. **配置 Runner 为服务**：
   ```bash
   # 安装服务
   sudo ./svc.sh install
   
   # 启动服务
   sudo ./svc.sh start
   
   # 查看服务状态
   sudo ./svc.sh status
   ```

5. **验证 Runner 状态**：
   - 返回 Gitea 仓库的 **Actions** → **Runners** 页面
   - 查看 Runner 是否显示为 **在线** 状态

#### 4.2.3 配置 Runner 环境

1. **安装必要依赖**：
   ```bash
   # 更新包索引
   sudo apt update
   
   # 安装必要依赖
   sudo apt install -y \
     build-essential \
     curl \
     git \
     jq \
     libssl-dev \
     libffi-dev \
     python3 \
     python3-pip \
     python3-venv \
     openjdk-17-jdk \
     maven \
     gradle
   ```

2. **配置 Docker 访问**：
   ```bash
   # 将 Runner 用户添加到 docker 组
   sudo usermod -aG docker gitea-runner
   
   # 重启 Docker 服务
   sudo systemctl restart docker
   ```

## 5. CI/CD 工作流示例

### 5.1 Maven 项目示例

#### 5.1.1 初始化 Maven 项目

```bash
# 创建项目目录
mkdir -p demo-project
cd demo-project

# 使用 Maven 初始化项目
mvn archetype:generate -DgroupId=com.example -DartifactId=demo-project -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.4 -DinteractiveMode=false

# 进入项目目录
cd demo-project
```

#### 5.1.2 项目结构

```
demo-project/
├── src/
│   ├── main/
│   │   └── java/
│   │       └── com/
│   │           └── example/
│   │               └── App.java
│   └── test/
│       └── java/
│           └── com/
│               └── example/
│                   └── AppTest.java
├── pom.xml
└── .gitea/workflows/
    └── ci-cd.yml
```

#### 5.1.3 配置 pom.xml

```bash
# 编辑 pom.xml 文件，添加 Nexus 部署配置
cat > pom.xml << EOF
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>demo-project</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <distributionManagement>
        <repository>
            <id>nexus-releases</id>
            <name>Nexus Release Repository</name>
            <url>http://<nexus-ip>:8081/repository/maven-releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id>
            <name>Nexus Snapshot Repository</name>
            <url>http://<nexus-ip>:8081/repository/maven-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
EOF
```

#### 5.1.4 配置 Maven  settings.xml

```bash
# 创建 Maven settings.xml 文件
mkdir -p ~/.m2
cat > ~/.m2/settings.xml << EOF
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>${env.MAVEN_USERNAME}</username>
            <password>${env.MAVEN_PASSWORD}</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>${env.MAVEN_USERNAME}</username>
            <password>${env.MAVEN_PASSWORD}</password>
        </server>
    </servers>
    
    <mirrors>
        <mirror>
            <id>nexus-public</id>
            <name>Nexus Public Mirror</name>
            <url>http://<nexus-ip>:8081/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
</settings>
EOF
```

#### 5.1.5 CI/CD 工作流配置

创建 `.gitea/workflows/ci-cd.yml` 文件：

```bash
# 创建 workflows 目录
mkdir -p .gitea/workflows

# 编辑工作流配置文件
cat > .gitea/workflows/ci-cd.yml << EOF
name: Maven CI/CD

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ main ]

jobs:
  build-test-deploy:
    runs-on: self-hosted
    steps:
    # 1. 检出代码
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    # 2. 设置 JDK 环境
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    
    # 3. 安装依赖
    - name: Install dependencies
      run: mvn install -DskipTests
    
    # 4. 运行单元测试
    - name: Run unit tests
      run: mvn test
    
    # 5. 构建项目
    - name: Build project
      run: mvn package -DskipTests
    
    # 6. 部署到 Nexus
    - name: Deploy to Nexus
      run: mvn deploy -DskipTests
      env:
        MAVEN_USERNAME: ${{ secrets.NEXUS_USERNAME }}
        MAVEN_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}
    
    # 7. 清理工作目录
    - name: Clean workspace
      run: mvn clean
EOF
```

### 5.2 Docker 项目示例

#### 5.2.1 初始化 Docker 项目

```bash
# 创建项目目录
mkdir -p docker-demo
cd docker-demo

# 创建简单的应用文件
echo 'FROM nginx:alpine
COPY index.html /usr/share/nginx/html/' > Dockerfile
echo '<h1>Hello from Docker CI/CD!</h1>' > index.html
```

#### 5.2.2 项目结构

```
docker-demo/
├── Dockerfile
├── index.html
└── .gitea/workflows/
    └── ci-cd.yml
```

#### 5.2.3 Dockerfile 示例

```dockerfile
FROM nginx:1.25.3-alpine
LABEL maintainer="demo@example.com"
LABEL version="1.0"
LABEL description="Docker CI/CD Demo"

COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

#### 5.2.4 CI/CD 工作流配置

创建 `.gitea/workflows/ci-cd.yml` 文件：

```bash
# 创建 workflows 目录
mkdir -p .gitea/workflows

# 编辑工作流配置文件
cat > .gitea/workflows/ci-cd.yml << EOF
name: Docker CI/CD

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ main ]

jobs:
  build-push:
    runs-on: self-hosted
    steps:
    # 1. 检出代码
    - name: Checkout code
      uses: actions/checkout@v4
    
    # 2. 设置 Docker Buildx
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    # 3. 登录到 Nexus Docker 仓库
    - name: Login to Nexus Docker Registry
      uses: docker/login-action@v3
      with:
        registry: http://<nexus-ip>:8082
        username: ${{ secrets.NEXUS_USERNAME }}
        password: ${{ secrets.NEXUS_PASSWORD }}
    
    # 4. 提取镜像元数据
    - name: Extract metadata (tags, labels)
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: <nexus-ip>:8082/demo/docker-demo
        tags: |
          type=ref,event=branch
          type=ref,event=tag
          type=sha,prefix=,suffix=
        labels: |
          org.opencontainers.image.title=Docker Demo
          org.opencontainers.image.description=Docker CI/CD Demo Project
          org.opencontainers.image.version={{raw}}
    
    # 5. 构建并推送 Docker 镜像
    - name: Build and push Docker image
      uses: docker/build-push-action@v6
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    # 6. 验证镜像（可选）
    - name: Verify Docker image
      if: github.event_name != 'pull_request'
      run: |
        docker pull <nexus-ip>:8082/demo/docker-demo:${{ github.sha }}
        docker inspect <nexus-ip>:8082/demo/docker-demo:${{ github.sha }}
    
    # 7. 清理本地镜像（可选）
    - name: Clean up local images
      if: github.event_name != 'pull_request'
      run: |
        docker rmi <nexus-ip>:8082/demo/docker-demo:${{ github.sha }} || true
EOF
```

## 6. 配置密钥

1. 进入 Gitea 仓库 → **设置** → **密钥**
2. 添加以下密钥：
   - `NEXUS_USERNAME`: Nexus 用户名
   - `NEXUS_PASSWORD`: Nexus 密码

## 7. 测试 CI/CD 工作流

1. 将代码推送到 Gitea 仓库的 `main` 分支
2. 进入仓库 → **Actions** 查看工作流运行状态
3. 检查 Nexus 仓库中是否成功上传了 artifacts 或 Docker 镜像

## 8. 常见问题和解决方案

### 8.1 Nexus 无法访问
- 检查 Nexus 容器是否正在运行：`docker ps`
- 检查防火墙设置，确保端口 8081、8082 等已开放
- 检查 Nexus 日志：`docker logs nexus`

### 8.2 Gitea Actions 运行失败
- 检查 Runner 是否在线：仓库 → **Actions** → **Runners**
- 检查工作流日志，查看具体错误信息
- 确保 Runner 服务器上已安装所需的依赖（如 Java、Docker 等）

### 8.3 无法推送到 Nexus Docker 仓库
- 检查 Docker 客户端配置，确保已添加 Nexus 仓库为不安全仓库（如果使用 HTTP）
  - 在 `/etc/docker/daemon.json` 中添加：
    ```json
    {
      "insecure-registries": ["<nexus-ip>:8082"]
    }
    ```
  - 重启 Docker 服务：`systemctl restart docker`
- 检查 Nexus Docker 仓库配置，确保 HTTP 端口已正确设置

## 9. 最佳实践

1. **代码质量检查**: 在 CI 工作流中添加代码质量检查工具，如 SonarQube
2. **安全扫描**: 添加依赖安全扫描，如 OWASP Dependency-Check
3. **多环境部署**: 配置不同环境（开发、测试、生产）的部署流程
4. **自动化测试**: 确保每个提交都运行自动化测试
5. **版本管理**: 使用语义化版本控制，自动生成版本号
6. **缓存优化**: 利用 Gitea Actions 缓存功能，加速构建过程
7. **通知机制**: 配置构建结果通知（如邮件、Slack 等）

## 10. 总结

通过本文的配置，您已经成功构建了基于 Nexus + Gitea + Gitea Actions 的 CI/CD 工作流。这个工作流可以帮助您自动化构建、测试和部署过程，提高开发效率和代码质量。

您可以根据自己的项目需求，修改和扩展这个工作流，添加更多的功能和步骤。
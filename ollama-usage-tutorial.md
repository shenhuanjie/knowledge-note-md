# Ollama 使用教程

## 1. Ollama 简介

Ollama 是一个用于本地运行、管理和开发大型语言模型（LLM）的开源工具，它允许用户在自己的计算机上轻松部署和使用各种大语言模型，无需复杂的配置和大量的计算资源。

### 1.1 Ollama 的主要特点

- **简单易用**：提供简洁的命令行界面，一键安装和运行模型
- **支持多种模型**：兼容 Llama 3、Mistral、Gemma 等主流开源模型
- **轻量级**：占用资源少，适合在普通电脑上运行
- **API 支持**：提供 REST API，方便集成到各种应用中
- **模型定制**：支持模型微调、合并和扩展
- **跨平台**：支持 Windows、macOS 和 Linux 系统

## 2. Ollama 安装

### 2.1 安装方式

#### macOS
```bash
# 使用 Homebrew 安装
brew install ollama
```

#### Linux
```bash
# 使用官方脚本安装
curl -fsSL https://ollama.com/install.sh | sh
```

#### Windows
从 [Ollama 官方网站](https://ollama.com/) 下载并安装 Windows 版本

### 2.2 验证安装

```bash
ollama --version
```

### 2.3 启动 Ollama 服务

大多数系统会自动启动 Ollama 服务，如需手动启动：

```bash
# Linux
sudo systemctl start ollama
sudo systemctl enable ollama

# macOS
sudo launchctl load /Library/LaunchDaemons/com.ollama.ollama.plist
```

## 3. Ollama 基本命令

### 3.1 查看帮助

```bash
ollama help
# 查看特定命令的帮助
ollama help run
```

### 3.2 模型管理

#### 3.2.1 查看可用模型

```bash
ollama list
```

#### 3.2.2 拉取模型

```bash
# 拉取默认版本
ollama pull llama3

# 拉取特定版本
ollama pull llama3:8b-instruct-q4_0

# 拉取其他模型
ollama pull mistral:7b
ollama pull gemma:2b
ollama pull qwen:7b
```

#### 3.2.3 删除模型

```bash
ollama rm llama3
```

#### 3.2.4 复制模型

```bash
ollama cp llama3 my-llama3
```

#### 3.2.5 查看模型信息

```bash
ollama show llama3
```

## 4. 使用 Ollama 运行模型

### 4.1 交互式对话

```bash
# 启动交互式对话
ollama run llama3

# 在对话中使用命令
/help    # 显示帮助信息
/bye     # 退出对话
/reset   # 重置对话历史
/set     # 设置参数（如 temperature、top_p 等）

# 示例：设置温度参数
/set temperature 0.7
```

### 4.2 单次生成

```bash
# 直接生成响应
ollama run llama3 "你好，Ollama！"

# 将输出保存到文件
ollama run llama3 "写一首关于春天的诗" > spring-poem.txt
```

### 4.3 使用自定义提示

```bash
# 使用 -p 参数指定提示
ollama run llama3 -p "作为一名技术专家，请解释：{prompt}" "什么是机器学习？"
```

### 4.4 批量处理

```bash
# 从文件读取提示并生成响应
cat prompts.txt | ollama run llama3 > responses.txt
```

## 5. 模型参数调整

### 5.1 常用参数

| 参数 | 描述 | 默认值 | 范围 |
|------|------|--------|------|
| temperature | 控制输出的随机性 | 0.8 | 0.0-1.0 |
| top_p | 控制输出的多样性（核采样） | 0.9 | 0.0-1.0 |
| top_k | 控制考虑的词汇数量 | 40 | 1-100 |
| num_ctx | 上下文窗口大小 | 2048 | 1-8192+ |
| num_predict | 最大生成 tokens 数 | 128 | 1-∞ |
| repeat_penalty | 重复惩罚 | 1.1 | 1.0-2.0 |

### 5.2 调整参数的方式

#### 5.2.1 在对话中使用 /set 命令

```bash
ollama run llama3
/set temperature 0.5
/set num_predict 512
```

#### 5.2.2 在命令行中使用参数

```bash
ollama run llama3 --temperature 0.5 --num-predict 512 "写一篇关于人工智能的短文"
```

#### 5.2.3 创建自定义模型文件

创建一个 `Modelfile` 文件：

```
FROM llama3
PARAMETER temperature 0.7
PARAMETER num_ctx 4096
SYSTEM "你是一个友好的AI助手，总是给出详细而有用的回答。"
```

然后创建自定义模型：

```bash
ollama create my-llama3 -f Modelfile
```

使用自定义模型：

```bash
ollama run my-llama3
```

## 6. 使用 Ollama API

### 6.1 API 端点

Ollama 默认在 `http://localhost:11434` 提供 API 服务

### 6.2 常用 API 调用

#### 6.2.1 拉取模型

```bash
curl http://localhost:11434/api/pull -d '{"name": "llama3"}'
```

#### 6.2.2 生成响应

```bash
# 基本生成
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "你好，Ollama！"}'

# 带参数的生成
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "写一首关于秋天的诗", "temperature": 0.7, "num_predict": 512}'
```

#### 6.2.3 对话

```bash
curl http://localhost:11434/api/chat -d '{"model": "llama3", "messages": [{"role": "user", "content": "你好，Ollama！"}, {"role": "assistant", "content": "你好！我是 Ollama，很高兴为你服务。"}, {"role": "user", "content": "什么是 Ollama？"}]}'
```

#### 6.2.4 列出模型

```bash
curl http://localhost:11434/api/tags
```

#### 6.2.5 删除模型

```bash
curl http://localhost:11434/api/delete -d '{"name": "llama3"}'
```

### 6.3 API 响应格式

Ollama API 支持两种响应格式：

1. **流式响应**（默认）：逐块返回生成结果，适合实时显示
2. **非流式响应**：一次性返回完整结果

设置非流式响应：

```bash
curl http://localhost:11434/api/generate -d '{"model": "llama3", "prompt": "你好", "stream": false}'
```

## 7. Ollama Web UI

### 7.1 安装和运行 Web UI

```bash
docker run -d -p 3000:8080 --name ollama-webui -e OLLAMA_API_BASE_URL=http://localhost:11434/api ghcr.io/ollama-webui/ollama-webui:main
```

### 7.2 访问 Web UI

在浏览器中访问：http://localhost:3000

### 7.3 Web UI 主要功能

- 模型管理（拉取、删除、查看）
- 交互式对话
- 提示词模板管理
- 对话历史记录
- 参数调整
- API 密钥管理

## 8. 模型定制和扩展

### 8.1 使用 Modelfile 定制模型

Modelfile 是 Ollama 用于定义模型的配置文件，支持多种指令：

```
# 基础模型
FROM llama3

# 设置参数
PARAMETER temperature 0.7
PARAMETER num_ctx 4096

# 设置系统提示
SYSTEM "你是一个专业的软件工程师，擅长解释复杂的技术概念。"

# 添加模板
TEMPLATE "{{ if .System }}<system>{{ .System }}</system>{{ end }}{{ if .Prompt }}<user>{{ .Prompt }}</user>{{ end }}<assistant>{{ .Response }}"

# 添加文件
FILE ./data/knowledge.txt

# 微调指令
ADAPTER ./adapter.npz
```

### 8.2 创建自定义模型

```bash
ollama create my-engineer-llama -f Modelfile
```

### 8.3 微调模型

Ollama 支持使用 LoRA（Low-Rank Adaptation）进行模型微调：

```bash
# 使用示例数据微调模型
ollama finetune llama3 -d ./training-data.jsonl
```

训练数据格式（JSONL）：

```jsonl
{"prompt": "什么是 Python？", "response": "Python 是一种高级、解释型、通用的编程语言。"}
{"prompt": "Python 的主要特点是什么？", "response": "Python 具有简单易学、可读性强、面向对象、动态类型、丰富的库等特点。"}
```

## 9. Ollama 最佳实践

### 9.1 性能优化

1. **选择合适的模型大小**：根据硬件资源选择合适大小的模型（如 7B、13B、70B）
2. **调整上下文窗口**：根据需要调整 `num_ctx` 参数，避免不必要的资源消耗
3. **使用量化模型**：选择量化版本的模型（如 q4_0、q5_1），减少内存占用
4. **关闭不必要的服务**：在运行大型模型时，关闭其他占用资源的应用
5. **使用 GPU 加速**：如果有 NVIDIA GPU，确保启用 CUDA 加速

### 9.2 模型选择建议

| 使用场景 | 推荐模型 | 特点 |
|----------|----------|------|
| 一般对话 | Llama 3 8B | 平衡的性能和质量 |
| 代码生成 | Llama 3 70B Code | 出色的代码理解和生成能力 |
| 创意写作 | Mistral 7B | 流畅的语言生成 |
| 轻量级应用 | Gemma 2B | 占用资源少，适合边缘设备 |
| 多语言支持 | Qwen 7B | 优秀的中文支持 |

### 9.3 安全注意事项

1. **不要运行不可信的模型**：只使用来自可信来源的模型
2. **限制 API 访问**：如果在公共网络上使用，确保配置适当的防火墙规则
3. **注意数据隐私**：避免将敏感数据输入到模型中，尤其是在使用第三方模型时
4. **定期更新 Ollama**：保持 Ollama 版本最新，以获取安全修复和性能改进

## 10. 常见问题处理

### 10.1 模型拉取失败

- 检查网络连接
- 尝试使用代理
- 检查磁盘空间
- 验证模型名称是否正确

### 10.2 模型运行缓慢

- 关闭其他占用资源的应用
- 降低模型大小或使用量化版本
- 调整 `num_ctx` 参数
- 确保启用了硬件加速

### 10.3 内存不足错误

- 选择更小的模型
- 增加系统内存（如果可能）
- 调整 `num_ctx` 参数为较小值
- 使用量化程度更高的模型（如 q4_0）

### 10.4 API 无法访问

- 检查 Ollama 服务是否正在运行
- 验证端口是否正确（默认 11434）
- 检查防火墙设置
- 验证 API 请求格式是否正确

## 11. 高级功能

### 11.1 模型合并

Ollama 支持将多个模型合并为一个：

```bash
# 合并模型
ollama merge llama3 mistral -o merged-model
```

### 11.2 导出和导入模型

```bash
# 导出模型
ollama export llama3 ./llama3.gguf

# 导入模型
ollama import my-model ./llama3.gguf
```

### 11.3 使用自定义 GGUF 模型

```bash
# 将自定义 GGUF 模型导入 Ollama
ollama create my-custom-model -f ./model.gguf
```

## 12. 总结

Ollama 是一个功能强大且易于使用的工具，它为用户提供了在本地运行和管理大型语言模型的能力。通过本教程，您应该已经掌握了 Ollama 的基本使用方法，包括模型安装、运行、参数调整、API 调用和模型定制等。

随着大语言模型技术的不断发展，Ollama 也在持续更新和改进，为用户提供更好的体验和更多的功能。建议您定期查看 Ollama 的官方文档和更新日志，以了解最新的功能和最佳实践。

希望本教程对您有所帮助，祝您在使用 Ollama 的过程中取得愉快的体验！

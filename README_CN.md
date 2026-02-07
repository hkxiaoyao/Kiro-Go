# Kiro-Go

[![Go Version](https://img.shields.io/badge/Go-1.21+-00ADD8?style=flat&logo=go)](https://go.dev/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat&logo=docker)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

将 Kiro 账号转换为 OpenAI / Anthropic 兼容的 API 服务。

[English](README.md) | 中文

## 功能特性

- 🔄 **Anthropic Claude API** - 完整支持 `/v1/messages` 端点
- 🤖 **OpenAI Chat API** - 兼容 `/v1/chat/completions`
- ⚖️ **多账号池** - 轮询负载均衡
- 🔐 **自动刷新 Token** - 无缝 Token 管理
- 📡 **流式响应** - 实时 SSE 输出
- 🎛️ **Web 管理面板** - 便捷的账号管理
- 🔑 **多种认证方式** - AWS Builder ID、IAM Identity Center (企业 SSO)、SSO Token、本地缓存、凭证 JSON
- 📊 **用量追踪** - 监控请求数、Token、Credits

## 快速开始

### Docker Compose（推荐）

```bash
git clone https://github.com/Quorinex/Kiro-Go.git
cd Kiro-Go

# 创建数据目录用于持久化
mkdir -p data

docker-compose up -d
```

### Docker 运行

```bash
# 创建数据目录
mkdir -p /path/to/data

docker run -d \
  --name kiro-go \
  -p 8080:8080 \
  -e ADMIN_PASSWORD=your_secure_password \
  -v /path/to/data:/app/data \
  --restart unless-stopped \
  ghcr.io/quorinex/kiro-go:latest
```

> 📁 `/app/data` 卷存储 `config.json`（包含账号和设置），挂载此目录以实现数据持久化。

### 源码编译

```bash
git clone https://github.com/Quorinex/Kiro-Go.git
cd Kiro-Go
go build -o kiro-go .
./kiro-go
```

## 配置

首次运行会自动创建 `data/config.json`：

```json
{
  "password": "changeme",
  "port": 8080,
  "host": "127.0.0.1",
  "requireApiKey": false,
  "apiKey": "",
  "accounts": []
}
```

> ⚠️ **生产环境请务必修改默认密码！**

## 环境变量

| 变量 | 说明 | 默认值 |
|-----|------|-------|
| `CONFIG_PATH` | 配置文件路径 | `data/config.json` |
| `ADMIN_PASSWORD` | 管理面板密码（覆盖配置文件） | - |

## 使用方法

### 1. 访问管理面板

打开 `http://localhost:8080/admin`，输入密码登录。

### 2. 添加账号

支持多种方式：

| 方式 | 说明 |
|------|------|
| **AWS Builder ID** | 通过 AWS Builder ID 授权登录（个人账号） |
| **IAM Identity Center (企业 SSO) 登录** | 通过 IAM Identity Center (企业 SSO) 授权登录（企业账号） |
| **SSO Token** | 通过浏览器 `x-amz-sso_authn` Token 添加账号 |
| **Kiro 本地缓存** | 通过 Kiro IDE 本地缓存文件添加账号 |
| **凭证 JSON** | 通过 Kiro Account Manager 导出的凭证添加账号 |

#### 凭证格式

```json
{
  "refreshToken": "eyJ...",
  "accessToken": "eyJ...",
  "clientId": "xxx",
  "clientSecret": "xxx"
}
```

### 3. 调用 API

#### Claude API

```bash
curl http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "你好！"}]
  }'
```

#### OpenAI API

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer any" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "你好！"}]
  }'
```

## 模型映射

| 请求模型 | 实际模型 |
|---------|---------|
| `claude-sonnet-4-20250514` | claude-sonnet-4-20250514 |
| `claude-sonnet-4.5` | claude-sonnet-4.5 |
| `claude-haiku-4.5` | claude-haiku-4.5 |
| `claude-opus-4.5` | claude-opus-4.5 |
| `gpt-4o`, `gpt-4` | claude-sonnet-4-20250514 |
| `gpt-3.5-turbo` | claude-sonnet-4-20250514 |

## 思考模式

在模型名称后添加后缀（默认：`-thinking`）即可启用扩展思考模式。

### 使用方法

```bash
# OpenAI API 启用思考
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4.5-thinking",
    "messages": [{"role": "user", "content": "一步步解决：15 * 23"}],
    "stream": true
  }'

# Claude API 启用思考
curl http://localhost:8080/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4.5-thinking",
    "max_tokens": 4096,
    "messages": [{"role": "user", "content": "分析这个问题"}]
  }'
```

### 配置

在管理面板的 **设置 > Thinking 模式设置** 中配置：

| 设置 | 说明 | 选项 |
|-----|------|------|
| **触发后缀** | 启用思考的模型名称后缀 | 默认：`-thinking`（可自定义，如 `-think`、`-sikao`） |
| **OpenAI 输出格式** | OpenAI API 中思考内容的返回方式 | `reasoning_content`（DeepSeek 兼容）、`<thinking>` 标签、`<think>` 标签 |
| **Claude 输出格式** | Claude API 中思考内容的返回方式 | `<thinking>` 标签（默认）、`<think>` 标签、纯文本 |

### 输出格式说明

**OpenAI API (`/v1/chat/completions`)**：
- `reasoning_content` - 思考内容放在单独的 `reasoning_content` 字段（DeepSeek 兼容）
- `thinking` - 思考内容用 `<thinking>...</thinking>` 标签包裹在 content 中
- `think` - 思考内容用 `<think>...</think>` 标签包裹在 content 中

**Claude API (`/v1/messages`)**：
- `thinking` - 思考内容用 `<thinking>...</thinking>` 标签包裹（默认）
- `think` - 思考内容用 `<think>...</think>` 标签包裹
- `reasoning_content` - 纯文本输出

## API 端点

| 端点 | 说明 |
|-----|------|
| `GET /health` | 健康检查 |
| `GET /v1/models` | 模型列表 |
| `POST /v1/messages` | Claude Messages API |
| `POST /v1/messages/count_tokens` | Token 计数 |
| `POST /cc/v1/messages` | Claude Code 流式（缓冲后返回，确保 input_tokens 准确） |
| `POST /cc/v1/messages/count_tokens` | Token 计数（Claude Code） |
| `POST /v1/chat/completions` | OpenAI Chat API |
| `GET /admin` | 管理面板 |

> `/cc/v1/messages` 会缓冲上游事件，在返回 `message_start` 时写入最终 `input_tokens`。
> 缓冲期间每 25 秒发送一次 `ping` 事件用于保活。

## 项目结构

```
Kiro-Go/
├── main.go              # 入口
├── config/              # 配置管理
├── pool/                # 账号池 & 负载均衡
├── proxy/               # API 处理 & Kiro 客户端
│   ├── handler.go       # HTTP 路由 & 管理 API
│   ├── kiro.go          # Kiro API 客户端
│   ├── kiro_api.go      # Kiro REST API（用量、模型）
│   └── translator.go    # 请求/响应转换
├── auth/                # 认证
│   ├── builderid.go     # AWS Builder ID 登录
│   ├── iam_sso.go       # IAM SSO 登录
│   ├── oidc.go          # OIDC Token 刷新
│   └── sso_token.go     # SSO Token 导入
├── web/                 # 管理面板前端
├── Dockerfile
└── docker-compose.yml
```

## 免责声明

本项目仅供**学习和研究目的**使用。

- 本软件与 Amazon、AWS 或 Kiro 没有任何关联、认可或合作关系
- 用户需自行确保其使用行为符合所有适用的服务条款和法律法规
- 作者不对因使用本软件而产生的任何滥用或违规行为承担责任
- 使用风险自负

使用本软件即表示您已阅读并理解本免责声明。

## 许可证

[MIT](LICENSE)

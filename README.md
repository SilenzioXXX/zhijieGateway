# Kong AI Gateway 项目

基于 Kong Gateway 搭建的本地 AI 模型网关，支持负载均衡、限流、认证、日志等功能。

## 📋 项目结构

```
kong/
├── docker-compose.yml      # Docker 编排配置
├── .env                     # 环境变量配置
├── config/
│   └── kong.yml            # Kong 声明式配置文件
├── logs/                    # 日志目录（自动生成）
└── README.md               # 项目文档
```

## 🚀 快速开始

### 前置要求

- Docker Desktop 已安装并运行
- 本地 AI 模型服务已启动（如 Ollama、LocalAI 等）

### 1. 启动 Kong Gateway

```bash
# 启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f kong
```

### 2. 等待服务启动

首次启动需要初始化数据库，大约需要 30 秒。等待所有服务健康检查通过。

### 3. 访问 Kong Manager（管理界面）

打开浏览器访问：http://localhost:8002

这是 Kong 的 Web 管理界面，可以可视化管理服务、路由、插件等。

### 4. 导入配置

```bash
# 方式一：使用声明式配置（推荐用于开发环境）
# 编辑 config/kong.yml 后，重启 Kong
docker-compose restart kong

# 方式二：使用 Kong Admin API
# 查看当前配置
curl -i http://localhost:8001/

# 查看服务列表
curl -i http://localhost:8001/services

# 查看路由列表
curl -i http://localhost:8001/routes
```

## 🔧 配置说明

### 端口说明

| 端口 | 用途 | 说明 |
|------|------|------|
| 8000 | HTTP Proxy | AI 模型请求入口 |
| 8443 | HTTPS Proxy | 安全代理端口 |
| 8001 | Admin API | Kong 管理 API |
| 8002 | Kong Manager | Web 管理界面 |
| 5432 | PostgreSQL | 数据库端口 |

### 本地 AI 模型配置

修改 `.env` 文件中的配置：

```bash
# 如果使用 Ollama（默认端口 11434）
LOCAL_AI_MODEL_HOST=host.docker.internal
LOCAL_AI_MODEL_PORT=11434

# 如果使用 LocalAI（默认端口 8080）
LOCAL_AI_MODEL_HOST=host.docker.internal
LOCAL_AI_MODEL_PORT=8080
```

然后修改 `config/kong.yml` 中的 `targets` 配置。

## 📡 API 使用示例

### 1. 测试 AI 聊天接口（带认证）

```bash
# 使用 API Key 认证
curl -X POST http://localhost:8000/api/chat \
  -H "X-API-Key: test-api-key-123456" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama2",
    "messages": [
      {
        "role": "user",
        "content": "你好，介绍一下自己"
      }
    ]
  }'
```

### 2. 获取模型列表

```bash
curl http://localhost:8000/api/models
```

### 3. 查看限流信息

响应头中会包含限流信息：
- `X-RateLimit-Limit-Minute`: 每分钟限制数
- `X-RateLimit-Remaining-Minute`: 剩余请求数

## 🔐 API Key 管理

### 默认 API Key

开发环境提供了两个测试 API Key：
- 普通用户：`test-api-key-123456`
- 管理员：`admin-api-key-abcdef`

### 添加新的 API Key

```bash
# 1. 创建消费者（用户）
curl -X POST http://localhost:8001/consumers \
  -d "username=new-user" \
  -d "custom_id=user-001"

# 2. 为消费者创建 API Key
curl -X POST http://localhost:8001/consumers/new-user/key-auth \
  -d "key=your-new-api-key"
```

或在 Kong Manager 界面中创建（更直观）。

## ⚡ 性能优化

### 1. 负载均衡

在 `config/kong.yml` 的 `upstreams.targets` 中添加多个模型实例：

```yaml
targets:
  - target: host.docker.internal:11434
    weight: 100
  - target: host.docker.internal:11435
    weight: 100
```

### 2. 调整限流配置

根据实际情况修改 `rate-limiting` 插件的配置：

```yaml
- name: rate-limiting
  config:
    minute: 60    # 根据模型性能调整
    hour: 1000
```

### 3. 超时时间配置

AI 生成响应可能较慢，已配置较长超时：
- `connect_timeout`: 60 秒
- `write_timeout`: 300 秒（5 分钟）
- `read_timeout`: 300 秒

## 📊 监控和日志

### 查看 Kong 日志

```bash
# 实时查看所有日志
docker-compose logs -f

# 只查看 Kong 日志
docker-compose logs -f kong

# 查看 AI 聊天日志
docker exec -it kong-gateway tail -f /usr/local/kong/logs/ai-chat.log
```

### 查看服务健康状态

```bash
# Kong 健康检查
curl http://localhost:8001/status

# 数据库连接状态
docker-compose exec kong-database pg_isready -U kong
```

## 🛠️ 常用命令

```bash
# 启动服务
docker-compose up -d

# 停止服务
docker-compose down

# 重启 Kong（配置修改后）
docker-compose restart kong

# 查看服务状态
docker-compose ps

# 进入 Kong 容器
docker exec -it kong-gateway bash

# 清理所有数据（谨慎使用）
docker-compose down -v
```

## 🔍 故障排查

### 1. Kong 无法启动

检查数据库是否正常：
```bash
docker-compose logs kong-database
```

### 2. 无法连接到本地 AI 模型

确认：
- AI 模型服务正在运行
- 端口配置正确
- 使用 `host.docker.internal` 而非 `localhost`

测试连接：
```bash
docker exec -it kong-gateway curl http://host.docker.internal:11434/api/tags
```

### 3. API Key 认证失败

检查：
- 请求头是否包含 `X-API-Key`
- API Key 是否正确
- 消费者是否已创建

## 📚 扩展功能

### 1. 添加 HTTPS 支持

生成自签名证书：
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./config/kong.key \
  -out ./config/kong.crt
```

在 Kong Manager 中配置证书。

### 2. 集成 Prometheus 监控

添加 Prometheus 插件：
```bash
curl -X POST http://localhost:8001/plugins \
  -d "name=prometheus"
```

访问指标：http://localhost:8001/metrics

### 3. 添加更多 AI 服务

复制 `ai-chat-service` 配置，修改路由路径和上游地址即可。

## 📖 相关文档

- [Kong Gateway 官方文档](https://docs.konghq.com/)
- [Kong 插件文档](https://docs.konghq.com/hub/)
- [Docker Compose 文档](https://docs.docker.com/compose/)

## 🤝 技术支持

如有问题，可以：
1. 查看 Kong 日志：`docker-compose logs kong`
2. 访问 Kong Admin API：http://localhost:8001
3. 使用 Kong Manager 界面排查

---

**注意**：这是开发环境配置，生产环境需要：
- 使用强密码和安全的 API Key
- 配置 HTTPS
- 使用外部数据库
- 配置防火墙和安全组
- 启用日志聚合和监控

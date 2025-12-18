# GeminiCLI to API

## 安装指南

### Docker 环境

**Docker 运行命令**
```bash
# 使用通用密码
docker run -d --name gcli2api --network host -e PASSWORD=pwd -e PORT=7861 -v $(pwd)/data/creds:/app/creds ghcr.io/su-kaka/gcli2api:latest

# 使用分离密码
docker run -d --name gcli2api --network host -e API_PASSWORD=api_pwd -e PANEL_PASSWORD=panel_pwd -e PORT=7861 -v $(pwd)/data/creds:/app/creds ghcr.io/su-kaka/gcli2api:latest
```

**Docker Compose 运行命令**
1. 将以下内容保存为 `docker-compose.yml` 文件：
    ```yaml
    version: '3.8'
    
    services:
      gcli2api:
        image: ghcr.io/su-kaka/gcli2api:latest
        container_name: gcli2api
        restart: unless-stopped
        network_mode: host
        environment:
          # 使用通用密码（推荐用于简单部署）
          - PASSWORD=pwd
          - PORT=7861
          # 或使用分离密码（推荐用于生产环境）
          # - API_PASSWORD=your_api_password
          # - PANEL_PASSWORD=your_panel_password
        volumes:
          - ./data/creds:/app/creds
        healthcheck:
          test: ["CMD-SHELL", "python -c \"import sys, urllib.request, os; port = os.environ.get('PORT', '7861'); req = urllib.request.Request(f'http://localhost:{port}/v1/models', headers={'Authorization': 'Bearer ' + os.environ.get('PASSWORD', 'pwd')}); sys.exit(0 if urllib.request.urlopen(req, timeout=5).getcode() == 200 else 1)\""]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 40s
    ```
2. 启动服务：
    ```bash
    docker-compose up -d
    ```

---

## ⚠️ 注意事项

- 当前 OAuth 验证流程**仅支持本地主机（localhost）访问**，即须通过 `http://127.0.0.1:7861/auth` 完成认证（默认端口 7861，可通过 PORT 环境变量修改）。
- **如需在云服务器或其他远程环境部署，请先在本地运行服务并完成 OAuth 验证，获得生成的 json 凭证文件（位于 `./geminicli/creds` 目录）后，再在auth面板将该文件上传即可。**
- **请严格遵守使用限制，仅用于个人学习和非商业用途**

---

## 配置说明

1. 访问 `http://127.0.0.1:7861/auth` （默认端口，可通过 PORT 环境变量修改）
2. 完成 OAuth 认证流程（默认密码：`pwd`，可通过环境变量修改）
3. 配置客户端：

**OpenAI 兼容客户端：**
   - **端点地址**：`http://127.0.0.1:7861/v1`
   - **API 密钥**：`pwd`（默认值，可通过 API_PASSWORD 或 PASSWORD 环境变量修改）

**Gemini 原生客户端：**
   - **端点地址**：`http://127.0.0.1:7861`
   - **认证方式**：
     - `Authorization: Bearer your_api_password`
     - `x-goog-api-key: your_api_password` 
     - URL 参数：`?key=your_api_password`

## 💾 分布式存储模式

### 🌟 存储后端优先级

gcli2api 支持多种存储后端，按优先级自动选择：**Redis > Postgres > MongoDB > 本地文件**

### ⚡ Redis 分布式存储模式

### ⚙️ 启用 Redis 模式

**步骤 1: 配置 Redis 连接**
```bash
# 本地 Redis
export REDIS_URI="redis://localhost:6379"

# 带密码的 Redis
export REDIS_URI="redis://:password@localhost:6379"

# SSL 连接（推荐生产环境）
export REDIS_URI="rediss://default:password@host:6380"

# Upstash Redis（免费云服务）
export REDIS_URI="rediss://default:token@your-host.upstash.io:6379"

# 可选：自定义数据库索引（默认: 0）
export REDIS_DATABASE="1"
```

**步骤 2: 启动应用**
```bash
# 应用会自动检测 Redis 配置并优先使用 Redis 存储
python web.py
```

### 🐘 Postgres 分布式存储模式

如果未配置 Redis，或者你希望使用关系型数据库作为主要存储方案，gcli2api 也支持 Postgres（位于 Redis 之后，优先于 MongoDB）。

⚙️ 启用 Postgres 模式

步骤 1: 配置 Postgres 连接
```bash
# 使用标准 DSN（示例）
export POSTGRES_DSN="postgresql://user:password@localhost:5432/gcli2api"

# 也可以使用 socket 或其他 DSN 格式，取决于你的部署方式
```

步骤 2: 启动应用
```bash
# 应用会自动检测 POSTGRES_DSN 并在 Redis 未启用时优先使用 Postgres 存储
python web.py
```

### 🍃 MongoDB 分布式存储模式

### 🌟 备选存储方案

如果未配置 Redis，gcli2api 将尝试使用 **MongoDB 存储模式**，

### ⚙️ 启用 MongoDB 模式

**步骤 1: 配置 MongoDB 连接**
```bash
# 本地 MongoDB
export MONGODB_URI="mongodb://localhost:27017"

# MongoDB Atlas 云服务
export MONGODB_URI="mongodb+srv://username:password@cluster.mongodb.net"

# 带认证的 MongoDB
export MONGODB_URI="mongodb://admin:password@localhost:27017/admin"

# 可选：自定义数据库名称（默认: gcli2api）
export MONGODB_DATABASE="my_gcli_db"
```

**步骤 2: 启动应用**
```bash
# 应用会自动检测 MongoDB 配置并使用 MongoDB 存储
python web.py
```

**Docker 环境使用 MongoDB**
```bash
# 单机 MongoDB 部署
docker run -d --name gcli2api \
  -e MONGODB_URI="mongodb://mongodb:27017" \
  -e API_PASSWORD=your_password \
  --network your_network \
  ghcr.io/su-kaka/gcli2api:latest

# 使用 MongoDB Atlas
docker run -d --name gcli2api \
  -e MONGODB_URI="mongodb+srv://user:pass@cluster.mongodb.net/gcli2api" \
  -e API_PASSWORD=your_password \
  -p 7861:7861 \
  ghcr.io/su-kaka/gcli2api:latest
```

**Docker Compose 示例**
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: gcli2api-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password123
    volumes:
      - mongodb_data:/data/db
    ports:
      - "27017:27017"

  gcli2api:
    image: ghcr.io/su-kaka/gcli2api:latest
    container_name: gcli2api
    restart: unless-stopped
    depends_on:
      - mongodb
    environment:
      - MONGODB_URI=mongodb://admin:password123@mongodb:27017/admin
      - MONGODB_DATABASE=gcli2api
      - API_PASSWORD=your_api_password
      - PORT=7861
    ports:
      - "7861:7861"

volumes:
  mongodb_data:
```


### 🔧 高级配置

**MongoDB 连接优化**
```bash
# 连接池和超时配置
export MONGODB_URI="mongodb://localhost:27017?maxPoolSize=10&serverSelectionTimeoutMS=5000"

# 副本集配置
export MONGODB_URI="mongodb://host1:27017,host2:27017,host3:27017/gcli2api?replicaSet=myReplicaSet"

# 读写分离配置
export MONGODB_URI="mongodb://localhost:27017/gcli2api?readPreference=secondaryPreferred"
```

## 🏗️ 技术架构

### 环境变量配置

**基础配置**
- `PORT`: 服务端口（默认：7861）
- `HOST`: 服务器监听地址（默认：0.0.0.0）

**密码配置**
- `API_PASSWORD`: 聊天 API 访问密码（默认：继承 PASSWORD 或 pwd）
- `PANEL_PASSWORD`: 控制面板访问密码（默认：继承 PASSWORD 或 pwd）  
- `PASSWORD`: 通用密码，设置后覆盖上述两个（默认：pwd）

**性能和稳定性配置**
- `CALLS_PER_ROTATION`: 每个凭证轮换前的调用次数（默认：10）
- `RETRY_429_ENABLED`: 启用 429 错误自动重试（默认：true）
- `RETRY_429_MAX_RETRIES`: 429 错误最大重试次数（默认：3）
- `RETRY_429_INTERVAL`: 429 错误重试间隔，秒（默认：1.0）
- `ANTI_TRUNCATION_MAX_ATTEMPTS`: 抗截断最大重试次数（默认：3）

**网络和代理配置**
- `PROXY`: HTTP/HTTPS 代理地址（格式：`http://host:port`）
- `OAUTH_PROXY_URL`: OAuth 认证代理端点
- `GOOGLEAPIS_PROXY_URL`: Google APIs 代理端点
- `METADATA_SERVICE_URL`: 元数据服务代理端点

**自动化配置**
- `AUTO_BAN`: 启用凭证自动封禁（默认：true）
- `AUTO_LOAD_ENV_CREDS`: 启动时自动加载环境变量凭证（默认：false）

**兼容性配置**
- `COMPATIBILITY_MODE`: 启用兼容性模式，将 system 消息转为 user 消息（默认：false）

**日志配置**
- `LOG_LEVEL`: 日志级别（DEBUG/INFO/WARNING/ERROR，默认：INFO）
- `LOG_FILE`: 日志文件路径（默认：gcli2api.log）

**存储配置（按优先级）**

**Redis 配置（最高优先级）**
- `REDIS_URI`: Redis 连接字符串（设置后启用 Redis 模式）
  - 本地：`redis://localhost:6379`
  - 带密码：`redis://:password@host:6379`
  - SSL：`rediss://default:password@host:6380`
- `REDIS_DATABASE`: Redis 数据库索引（0-15，默认：0）

**MongoDB 配置（第二优先级）**
- `MONGODB_URI`: MongoDB 连接字符串（设置后启用 MongoDB 模式）
- `MONGODB_DATABASE`: MongoDB 数据库名称（默认：gcli2api）

**凭证配置**

支持使用 `GCLI_CREDS_*` 环境变量导入多个凭证：

#### 凭证环境变量使用示例

**方式 1：编号格式**
```bash
export GCLI_CREDS_1='{"client_id":"your-client-id","client_secret":"your-secret","refresh_token":"your-token","token_uri":"https://oauth2.googleapis.com/token","project_id":"your-project"}'
export GCLI_CREDS_2='{"client_id":"...","project_id":"..."}'
```

**方式 2：项目名格式**
```bash
export GCLI_CREDS_myproject='{"client_id":"...","project_id":"myproject",...}'
export GCLI_CREDS_project2='{"client_id":"...","project_id":"project2",...}'
```

**启用自动加载**
```bash
export AUTO_LOAD_ENV_CREDS=true  # 程序启动时自动导入环境变量凭证
```

**Docker 使用示例**
```bash
# 使用通用密码
docker run -d --name gcli2api \
  -e PASSWORD=mypassword \
  -e PORT=8080 \
  -e GOOGLE_CREDENTIALS="$(cat credential.json | base64 -w 0)" \
  ghcr.io/su-kaka/gcli2api:latest

# 使用分离密码
docker run -d --name gcli2api \
  -e API_PASSWORD=my_api_password \
  -e PANEL_PASSWORD=my_panel_password \
  -e PORT=8080 \
  -e GOOGLE_CREDENTIALS="$(cat credential.json | base64 -w 0)" \
  ghcr.io/su-kaka/gcli2api:latest
```

注意：当设置了凭证环境变量时，系统将优先使用环境变量中的凭证，忽略 `creds` 目录中的文件。

### API 使用方式

本服务支持两套完整的 API 端点：

#### 1. OpenAI 兼容端点

**端点：** `/v1/chat/completions`  
**认证：** `Authorization: Bearer your_api_password`

支持两种请求格式，会自动检测并处理：

**OpenAI 格式：**
```json
{
  "model": "gemini-2.5-pro",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant"},
    {"role": "user", "content": "Hello"}
  ],
  "temperature": 0.7,
  "stream": true
}
```

**Gemini 原生格式：**
```json
{
  "model": "gemini-2.5-pro",
  "contents": [
    {"role": "user", "parts": [{"text": "Hello"}]}
  ],
  "systemInstruction": {"parts": [{"text": "You are a helpful assistant"}]},
  "generationConfig": {
    "temperature": 0.7
  }
}
```

#### 2. Gemini 原生端点

**非流式端点：** `/v1/models/{model}:generateContent`  
**流式端点：** `/v1/models/{model}:streamGenerateContent`  
**模型列表：** `/v1/models`

**认证方式（任选一种）：**
- `Authorization: Bearer your_api_password`
- `x-goog-api-key: your_api_password`  
- URL 参数：`?key=your_api_password`

**请求示例：**
```bash
# 使用 x-goog-api-key 头部
curl -X POST "http://127.0.0.1:7861/v1/models/gemini-2.5-pro:generateContent" \
  -H "x-goog-api-key: your_api_password" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [
      {"role": "user", "parts": [{"text": "Hello"}]}
    ]
  }'

# 使用 URL 参数
curl -X POST "http://127.0.0.1:7861/v1/models/gemini-2.5-pro:streamGenerateContent?key=your_api_password" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [
      {"role": "user", "parts": [{"text": "Hello"}]}
    ]
  }'
```

**Gemini 原生banana：**
```python
from io import BytesIO
from PIL import Image
from google.genai import Client
from google.genai.types import HttpOptions
from google.genai import types
# The client gets the API key from the environment variable `GEMINI_API_KEY`.

client = Client(
            api_key="pwd",
            http_options=HttpOptions(base_url="http://127.0.0.1:7861"),
        )

prompt = (
    """
    画一只猫
    """
)

response = client.models.generate_content(
    model="gemini-2.5-flash-image",
    contents=[prompt],
    config=types.GenerateContentConfig(
        image_config=types.ImageConfig(
            aspect_ratio="16:9",
        )
    )
)
for part in response.candidates[0].content.parts:
    if part.text is not None:
        print(part.text)
    elif part.inline_data is not None:
        image = Image.open(BytesIO(part.inline_data.data))
        image.save("generated_image.png")

```

**说明：**
- OpenAI 端点返回 OpenAI 兼容格式
- Gemini 端点返回 Gemini 原生格式
- 两种端点使用相同的 API 密码

### 聊天 API 功能特性

**多模态支持**
```json
{
  "model": "gemini-2.5-pro",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "描述这张图片"},
        {
          "type": "image_url",
          "image_url": {
            "url": "data:image/jpeg;base64,/9j/4AAQSkZJRgABA..."
          }
        }
      ]
    }
  ]
}
```

**思维模式支持**
```json
{
  "model": "gemini-2.5-pro-maxthinking",
  "messages": [
    {"role": "user", "content": "复杂数学问题"}
  ]
}
```

响应将包含分离的思维内容：
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "最终答案",
      "reasoning_content": "详细的思考过程..."
    }
  }]
}
```

**流式抗截断使用**
```json
{
  "model": "流式抗截断/gemini-2.5-pro",
  "messages": [
    {"role": "user", "content": "写一篇长文章"}
  ],
  "stream": true
}
```

**兼容性模式**
```bash
# 启用兼容性模式
export COMPATIBILITY_MODE=true
```
此模式下，所有 `system` 消息会转换为 `user` 消息，提高与某些客户端的兼容性。

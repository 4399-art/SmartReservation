# 智慧图书馆座位预约系统 - Docker 部署指南

## 项目概述

前后端分离的 AI 座位/空间预约平台。

- **前端**: Vue 3 + Vite + Element Plus → Nginx 镜像
- **后端**: FastAPI + SQLAlchemy + LangChain → Uvicorn 镜像
- **数据库**: MySQL（外部独立部署）

## 服务拓扑

```
浏览器 ──:80──► Nginx(前端) ──内部网络──► FastAPI(后端) ──:3306──► MySQL
                 │ 静态资源            │
                 └── /api/* ──────────┘
                 └── /uploads/* ──────┘
```

| 服务 | 容器内端口 | 对外端口 |
|------|-----------|---------|
| frontend (Nginx) | 80 | **80** |
| backend (FastAPI) | 8000 | 内部（仅 Nginx 可访问） |
| MySQL | 3306 | 外部独立服务器 |

---

## 前置条件

目标服务器需已安装：
- **Docker** + **Docker Compose v2**（`docker compose version` 能跑）
- **MySQL 8.x**（可在本机或其他服务器）

MySQL 需预先授权远程访问（在 MySQL 服务器上执行）：

```sql
CREATE DATABASE IF NOT EXISTS lab_agent DEFAULT CHARSET utf8mb4;
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

开放防火墙 80 端口：

```bash
firewall-cmd --permanent --add-port=80/tcp && firewall-cmd --reload
```

---

## 部署方式一：发送镜像给对方（推荐）

对方服务器**不需要源码**，只需要 Docker。

### 第一步：在构建机上打包镜像

```bash
cd SmartReservation

# 构建两个镜像
docker compose build

# 导出成 tar 文件
docker save smart-reservation-backend:latest -o backend.tar
docker save smart-reservation-frontend:latest -o frontend.tar
```

### 第二步：发给对方

把以下 **4 个文件** 发给对方：

```
backend.tar                       # 后端镜像
frontend.tar                      # 前端镜像
docker-compose.prod.yml           # 编排文件（纯运行，无 build）
backend/.env.example              # 配置模板（改名为 .env）
```

### 第三步：对方服务器上部署

```bash
# 1. 新建目录，导入镜像
mkdir -p /opt/deploy/backend
cd /opt/deploy
docker load -i backend.tar
docker load -i frontend.tar

# 2. 准备配置（从模板复制，改实际值）
cp /path/to/.env.example backend/.env
vi backend/.env
```

**⚠️ backend/.env 里必须改的项：**

```env
DATABASE_URL=mysql+pymysql://root:密码@MySQL_IP:3306/lab_agent
CORS_ORIGINS=http://服务器IP
LLM_API_KEY=你的key
LLM_BASE_URL=你的模型地址
LLM_MODEL=你的模型名
EMBEDDING_API_KEY=你的key
EMBEDDING_BASE_URL=你的嵌入地址
EMBEDDING_MODEL=你的嵌入模型名
```

```bash
# 3. 启动
cd /opt/deploy
docker compose -f docker-compose.prod.yml up -d

# 4. 验证
docker compose -f docker-compose.prod.yml ps
# 期望两个容器都是 healthy
# 浏览器访问 http://服务器IP/
```

---

## 部署方式二：服务器上构建（有源码）

```bash
# 1. 上传代码
scp -r SmartReservation root@服务器IP:/opt/

# 2. 改配置
cd /opt/SmartReservation
vi backend/.env       # 数据库、LLM 等

# 3. 构建启动
docker compose up -d --build

# 4. 验证
docker compose ps
```

---

## 配置项速查

### backend/.env（必须配）

| 变量 | 说明 | 示例 |
|------|------|------|
| `DATABASE_URL` | MySQL 连接串 | `mysql+pymysql://root:pwd@10.0.0.1:3306/lab_agent` |
| `CORS_ORIGINS` | 允许跨域的前端地址 | `http://192.168.120.128` 或 `*` |
| `LLM_API_KEY` | 大模型 Key | |
| `LLM_BASE_URL` | 大模型接口地址 | |
| `LLM_MODEL` | 大模型名称 | |
| `EMBEDDING_API_KEY` | 嵌入模型 Key | |
| `EMBEDDING_BASE_URL` | 嵌入接口地址 | |
| `EMBEDDING_MODEL` | 嵌入模型名称 | |

### docker-compose.prod.yml（可选改）

| 项 | 说明 | 默认 |
|----|------|------|
| `ports: "80:80"` | 前端对外端口 | 80 |
| `image:` | 镜像名 | `smart-reservation-*` |

### 前端 API 地址（无需配置）

前端已构建在镜像内，API 请求走**相对路径** `/api/*`，Nginx 自动代理到后端。**生产部署不需要单独配前端环境变量。**

---

## 目录结构（部署产物）

```
# 方式一（纯运行）
deploy/
├── docker-compose.prod.yml    # 编排文件
├── backend/
│   └── .env                   # ⚠️ 唯一要改的配置
└── (不需要其他任何文件)

# 方式二（带源码构建）
SmartReservation/
├── docker-compose.yml
├── docker-compose.prod.yml
├── backend/
│   ├── Dockerfile
│   ├── .env                   # ⚠️
│   ├── .env.example
│   ├── main.py / app/ / pyproject.toml
└── frontend/
    ├── Dockerfile
    ├── .dockerignore
    ├── docker/nginx.conf
    └── src/ / package.json
```

---

## 常用命令

```bash
# 指定 prod 文件启动/停止/重启
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml restart backend

# 构建模式才用 --build
docker compose up -d --build

# 查看状态和日志
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs -f --tail=200

# 进容器调试
docker compose -f docker-compose.prod.yml exec backend bash
docker compose -f docker-compose.prod.yml exec frontend sh

# 清理
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml down -v   # ⚠️ 连数据卷一起删
```

## 数据持久化

| Volume | 容器内路径 | 说明 |
|--------|-----------|------|
| `smart-reservation-uploads` | `/app/uploads` | 用户上传文件 |
| `smart-reservation-chroma` | `/app/chroma_db` | ChromaDB 向量库 |
| `smart-reservation-rag` | `/app/rag_uploads` | RAG 知识库文件 |

`docker compose down` **不会删数据卷**，只有加 `-v` 才会清。
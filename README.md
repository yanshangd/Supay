# Supay Docker 部署指南

## 环境要求

- Docker >= 20.10
- Docker Compose >= 2.0
- 建议系统：Linux / Windows(WSL2) / macOS

## 项目结构

```
supay-docker/
├── docker-compose.yml          # Docker Compose 编排文件
├── Dockerfile                  # 后端服务镜像构建文件
├── nginx.conf                  # Nginx 反向代理配置
├── Supay-springboot-1.0.0.jar  # 后端 Spring Boot 可执行 JAR
└── dist/                       # 前端构建产物（静态页面）
```

## 快速部署

### 1. 克隆或准备项目文件

确保以下文件已就位：
- `docker-compose.yml`
- `Dockerfile`
- `nginx.conf`
- `Supay-springboot-1.0.0.jar`
- `dist/` 

### 2. 启动服务

在项目根目录执行：

```bash
docker compose up -d
```

> Windows 环境如使用旧版 Docker Desktop，请使用 `docker-compose up -d`

### 3. 检查运行状态

```bash
docker compose ps
```

四个容器应均为 `running` 状态：
- `supay-wed`     — Nginx 前端代理
- `supay-backend` — Spring Boot 后端服务
- `supay-mysql`   — MySQL 5.7 数据库
- `supay-redis`   — Redis 缓存

### 4. 访问系统

- 前端页面：http://localhost
- 后端 API：http://localhost:8080
- MySQL 端口：3306（如宿主机已占用，请修改映射端口）
- Redis 端口：6379（如宿主机已占用，请修改映射端口）

## 服务说明

| 服务       | 镜像/构建方式              | 容器名           | 暴露端口 | 说明                     |
|------------|----------------------------|------------------|----------|--------------------------|
| wed        | `nginx:alpine`             | `supay-wed`      | 80       | 前端静态资源 & 反向代理  |
| backend    | 本地 Dockerfile 构建       | `supay-backend`  | 8080     | Spring Boot 业务后端     |
| mysql      | `mysql:5.7`                | `supay-mysql`    | 3306     | 业务数据库               |
| redis      | `redis:7-alpine`           | `supay-redis`    | 6379     | 缓存服务                 |

## 常用运维命令

```bash
# 启动服务（后台运行）
docker compose up -d

# 停止服务
docker compose down

# 停止并删除数据卷（谨慎操作，会清空数据库和上传文件）
docker compose down -v

# 查看日志
docker compose logs -f [service_name]

# 重启单个服务
docker compose restart backend

# 重新构建后端镜像
docker compose build backend

# 进入 MySQL 容器执行 SQL
docker exec -it supay-mysql mysql -usupay -psupay123 supay

# 进入 Redis 容器
docker exec -it supay-redis redis-cli
```

## 配置说明

### 数据库连接

后端默认通过 Docker 内部网络连接 MySQL：

```
地址：mysql:3306
数据库：supay
用户名：supay
密码：supay123
```

如需修改，请编辑 `docker-compose.yml` 中 `backend` 和 `mysql` 的对应环境变量，并保持一致。

### Redis 连接

```
地址：redis:6379
密码：无
```

如需设置 Redis 密码，请同步修改 `backend` 的 `SPRING_REDIS_PASSWORD`。

### 上传文件存储

上传文件持久化在 Docker 卷 `supay-uploads` 中，映射到容器内 `/app/uploads`。

### Nginx 路由规则

| 路径前缀   | 转发目标                     | 说明           |
|------------|------------------------------|----------------|
| `/`        | `/usr/share/nginx/html`      | 前端静态资源   |
| `/api/`    | `http://backend:8080`        | REST API 代理  |
| `/ws/`     | `http://backend:8080/ws/`    | WebSocket 代理 |
| `/uploads/`| `http://backend:8080/uploads/` | 文件访问代理   |

## 注意事项

1. **生产环境部署前**：
   - 务必修改所有默认密码（MySQL root、MySQL user、JWT_SECRET）。
   - 建议关闭 MySQL 和 Redis 的宿主机端口映射，或限制访问 IP。
   - 建议为 `APP_BASE_URL` 配置真实域名。

2. **数据库初始化**：
   - 首次启动 MySQL 会自动创建 `supay` 数据库。
   - 如需要导入初始 SQL，可将 SQL 文件挂载到 `/docker-entrypoint-initdb.d/`。

3. **前端更新**：
   - 直接替换 `dist/` 目录下的文件，然后重启 `wed` 容器即可：
     ```bash
     docker compose restart wed
     ```

4. **后端更新**：
   - 替换 `Supay-springboot-1.0.0.jar` 后重新构建并启动：
     ```bash
     docker compose up -d --build backend
     ```

5. **端口冲突**：
   - 若宿主机 80、8080、3306、6379 已被占用，请修改 `docker-compose.yml` 中的 `ports` 映射为其他端口，例如 `"8081:8080"。

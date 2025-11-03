# 集成与部署指南

> **开发环境搭建、CI/CD配置、生产部署完整流程**

---

## 📋 目录
- [本地开发环境](#本地开发环境)
- [Docker开发环境](#docker开发环境)
- [调试配置](#调试配置)
- [CI/CD流水线](#cicd流水线)
- [生产部署](#生产部署)
- [监控与日志](#监控与日志)

---

## 本地开发环境

### 1. 前置要求检查
```bash
# Node.js版本
node -v  # 需要 v20.0.0+

# PostgreSQL
psql --version  # 需要 15+

# Git
git --version

# Docker (可选)
docker --version
docker-compose --version
```

### 2. 克隆项目
```bash
git clone <your-repo-url>
cd teamos

# 项目结构
# teamos/
# ├── backend/          # NestJS后端
# ├── frontend/         # Next.js前端
# ├── docker-compose.yml
# └── README.md
```

### 3. 后端环境配置

#### 安装依赖
```bash
cd backend
npm install
```

#### 配置环境变量（backend/.env）
```bash
# 数据库
DATABASE_URL="postgresql://teamos:password@localhost:5432/teamos?schema=public"

# JWT
JWT_SECRET="your-super-secret-key-change-in-production-min-32-chars"
JWT_EXPIRES_IN="7d"

# 服务配置
NODE_ENV="development"
PORT=3000

# 前端URL（CORS）
FRONTEND_URL="http://localhost:3001"

# 日志级别
LOG_LEVEL="debug"
```

#### 初始化数据库
```bash
# 创建数据库
createdb teamos

# 或使用psql
psql -U postgres
CREATE DATABASE teamos;
\q

# 运行迁移
npx prisma migrate dev

# 查看数据库（可选）
npx prisma studio
```

#### 启动后端
```bash
npm run start:dev

# 验证
curl http://localhost:3000/api
# 应返回Swagger文档页面
```

### 4. 前端环境配置

#### 安装依赖
```bash
cd ../frontend
npm install
```

#### 配置环境变量（frontend/.env.local）
```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
```

#### 生成API类型
```bash
# 确保后端正在运行
npm run generate:api
```

#### 启动前端
```bash
npm run dev

# 访问 http://localhost:3001
```

---

## Docker开发环境

### 1. Docker Compose配置（docker-compose.yml）
```yaml
version: '3.8'

services:
  # PostgreSQL数据库
  postgres:
    image: postgres:15-alpine
    container_name: teamos-postgres
    environment:
      POSTGRES_USER: teamos
      POSTGRES_PASSWORD: password
      POSTGRES_DB: teamos
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U teamos"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 后端服务
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: development
    container_name: teamos-backend
    environment:
      DATABASE_URL: postgresql://teamos:password@postgres:5432/teamos?schema=public
      JWT_SECRET: dev-secret-key-change-in-production
      JWT_EXPIRES_IN: 7d
      NODE_ENV: development
      PORT: 3000
      FRONTEND_URL: http://localhost:3001
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app
      - /app/node_modules
    depends_on:
      postgres:
        condition: service_healthy
    command: npm run start:dev

  # 前端服务
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: development
    container_name: teamos-frontend
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:3000
    ports:
      - "3001:3001"
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    depends_on:
      - backend
    command: npm run dev

volumes:
  postgres_data:
```

### 2. 后端Dockerfile（backend/Dockerfile）
```dockerfile
# 开发阶段
FROM node:20-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
EXPOSE 3000
CMD ["npm", "run", "start:dev"]

# 构建阶段
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

# 生产阶段
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

# 复制依赖
COPY package*.json ./
RUN npm ci --omit=dev

# 复制构建产物
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/prisma ./prisma

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### 3. 前端Dockerfile（frontend/Dockerfile）
```dockerfile
# 开发阶段
FROM node:20-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3001
CMD ["npm", "run", "dev"]

# 依赖阶段
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json package-lock.json ./
RUN npm ci

# 构建阶段
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# 生产阶段
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

# 复制必要文件
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3001
CMD ["node", "server.js"]
```

### 4. 启动Docker环境
```bash
# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止服务
docker-compose down

# 重建镜像
docker-compose up -d --build
```

---

## 调试配置

### 1. VSCode调试配置（.vscode/launch.json）
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Backend",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "start:debug"],
      "cwd": "${workspaceFolder}/backend",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen",
      "skipFiles": ["<node_internals>/**"]
    },
    {
      "name": "Debug Frontend",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "cwd": "${workspaceFolder}/frontend",
      "console": "integratedTerminal",
      "serverReadyAction": {
        "pattern": "started server on .+, url: (https?://.+)",
        "uriFormat": "%s",
        "action": "debugWithChrome"
      }
    }
  ]
}
```

### 2. 后端调试脚本（backend/package.json）
```json
{
  "scripts": {
    "start:debug": "nest start --debug --watch"
  }
}
```

---

## CI/CD流水线

### 1. GitHub Actions工作流（.github/workflows/ci.yml）
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'

jobs:
  # 后端测试
  backend-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: teamos
          POSTGRES_PASSWORD: password
          POSTGRES_DB: teamos_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: backend
        run: npm ci

      - name: Generate Prisma Client
        working-directory: backend
        run: npx prisma generate

      - name: Run migrations
        working-directory: backend
        env:
          DATABASE_URL: postgresql://teamos:password@localhost:5432/teamos_test?schema=public
        run: npx prisma migrate deploy

      - name: Run tests
        working-directory: backend
        env:
          DATABASE_URL: postgresql://teamos:password@localhost:5432/teamos_test?schema=public
          JWT_SECRET: test-secret
        run: npm test

      - name: Run E2E tests
        working-directory: backend
        env:
          DATABASE_URL: postgresql://teamos:password@localhost:5432/teamos_test?schema=public
          JWT_SECRET: test-secret
        run: npm run test:e2e

  # 前端测试
  frontend-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: frontend
        run: npm ci

      - name: Run linter
        working-directory: frontend
        run: npm run lint

      - name: Run type check
        working-directory: frontend
        run: npm run type-check

      - name: Build
        working-directory: frontend
        run: npm run build

  # Docker构建
  docker-build:
    runs-on: ubuntu-latest
    needs: [backend-test, frontend-test]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/teamos-backend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/teamos-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 2. 代码质量检查（.github/workflows/code-quality.yml）
```yaml
name: Code Quality

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Lint backend
        working-directory: backend
        run: |
          npm ci
          npm run lint

      - name: Lint frontend
        working-directory: frontend
        run: |
          npm ci
          npm run lint
```

---

## 生产部署

### 1. 环境变量配置（生产环境）

#### backend/.env.production
```bash
DATABASE_URL="postgresql://user:password@prod-db:5432/teamos?schema=public"
JWT_SECRET="<生成强密钥: openssl rand -base64 32>"
JWT_EXPIRES_IN="7d"
NODE_ENV="production"
PORT=3000
FRONTEND_URL="https://teamos.example.com"
LOG_LEVEL="info"
```

#### frontend/.env.production
```bash
NEXT_PUBLIC_API_URL=https://api.teamos.example.com
```

### 2. Docker Compose生产配置（docker-compose.prod.yml）
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: teamos
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - teamos-network

  backend:
    image: your-registry/teamos-backend:latest
    restart: always
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/teamos?schema=public
      JWT_SECRET: ${JWT_SECRET}
      NODE_ENV: production
    ports:
      - "3000:3000"
    depends_on:
      - postgres
    networks:
      - teamos-network

  frontend:
    image: your-registry/teamos-frontend:latest
    restart: always
    environment:
      NEXT_PUBLIC_API_URL: https://api.teamos.example.com
    ports:
      - "3001:3001"
    depends_on:
      - backend
    networks:
      - teamos-network

  nginx:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - backend
      - frontend
    networks:
      - teamos-network

volumes:
  postgres_data:

networks:
  teamos-network:
    driver: bridge
```

### 3. Nginx配置（nginx.conf）
```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server backend:3000;
    }

    upstream frontend {
        server frontend:3001;
    }

    server {
        listen 80;
        server_name teamos.example.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name teamos.example.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        # API代理
        location /api {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # 前端代理
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### 4. 部署脚本（deploy.sh）
```bash
#!/bin/bash
set -e

echo "🚀 开始部署 TeamOS..."

# 拉取最新镜像
docker-compose -f docker-compose.prod.yml pull

# 停止旧容器
docker-compose -f docker-compose.prod.yml down

# 启动新容器
docker-compose -f docker-compose.prod.yml up -d

# 运行数据库迁移
docker-compose -f docker-compose.prod.yml exec -T backend npx prisma migrate deploy

# 健康检查
echo "⏳ 等待服务启动..."
sleep 10

if curl -f http://localhost:3000/api > /dev/null 2>&1; then
    echo "✅ 后端服务正常"
else
    echo "❌ 后端服务异常"
    exit 1
fi

if curl -f http://localhost:3001 > /dev/null 2>&1; then
    echo "✅ 前端服务正常"
else
    echo "❌ 前端服务异常"
    exit 1
fi

echo "🎉 部署完成！"
```

---

## 监控与日志

### 1. 日志配置（backend/src/logger.ts）
```typescript
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

export const loggerConfig = WinstonModule.createLogger({
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.colorize(),
        winston.format.printf(({ timestamp, level, message, context }) => {
          return `${timestamp} [${context}] ${level}: ${message}`;
        }),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
    new winston.transports.File({
      filename: 'logs/combined.log',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
    }),
  ],
});
```

### 2. 健康检查端点（backend/src/health/health.controller.ts）
```typescript
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, PrismaHealthIndicator } from '@nestjs/terminus';
import { PrismaService } from '../prisma/prisma.service';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prismaHealth: PrismaHealthIndicator,
    private prisma: PrismaService,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.prismaHealth.pingCheck('database', this.prisma),
    ]);
  }
}
```

---

**环境配置与部署流程已完成！继续阅读 [AI Agent集成](./04-agent-integration.md)** 🚀


# TeamOS 实施指南

> **个人级工程操作系统（TeamOS）完整构建指南**  
> 将"一个全栈PM/工程师 + 多个AI Agent"抽象成类似大厂的多团队协同模式

---

## 📚 文档导航

本指南提供从零开始到项目完成的全流程构建文档，涵盖技术选型、架构设计、开发实施、集成部署和AI Agent协作的完整路径。

### 核心文档

| 文档 | 描述 | 适用人群 |
|------|------|----------|
| [00-技术选型与架构](./00-tech-choices.md) | 技术栈选型理由、系统架构图、关键设计决策 | 架构师、技术负责人 |
| [01-后端实现指南](./01-backend.md) | NestJS + Prisma + OpenAPI 完整实现步骤 | 后端工程师 |
| [02-前端实现指南](./02-frontend.md) | Next.js + TanStack Query + UI组件库实现 | 前端工程师 |
| [03-集成与部署](./03-integration-deploy.md) | 开发环境、CI/CD、生产部署完整流程 | DevOps、全栈工程师 |
| [04-AI Agent集成](./04-agent-integration.md) | Agent角色定义、接口规范、协作流程 | AI工程师、集成工程师 |
| [05-ActivityStreams实现](./05-activitystreams.md) | AS2.0模型、扩展、存储与查询设计 | 后端工程师、架构师 |
| [06-OpenAPI工具链](./06-openapi-tooling.md) | Spectral校验、Redocly文档、客户端生成 | API设计师、全栈工程师 |

---

## 🎯 项目目标

### 核心理念
构建一个博客管理网站作为**单一事实来源（SSoT）**，通过标准化接口统一沉淀：
- **PRD（产品需求文档）**：需求定义与变更历史
- **UI设计稿**：设计资产与版本管理
- **Bug追踪**：缺陷报告与修复流程
- **代码构建**：构建状态与部署记录
- **决策日志**：架构决策与技术选型记录

### 预期价值
- **溯源**：任何变更都能回溯到需求→设计→实现→测试的完整链路
- **协同**：多个AI Agent通过标准化接口协作，模拟真实团队工作流
- **自动化**：从代码提交到版本发布的全流程自动化管理

---

## 🏗️ 技术架构概览

### 技术栈
- **前端**：Next.js 15 (App Router) + TanStack Query + shadcn/ui + Tailwind CSS 4
- **后端**：NestJS + Prisma (PostgreSQL) + OpenAPI 3.1
- **契约**：OpenAPI 3.1 + Spectral + Redocly
- **活动流**：ActivityStreams 2.0 (JSON-LD)
- **DevOps**：GitHub Actions + Docker + Docker Compose
- **AI Agent**：Microsoft AutoGen / CrewAI（基于OpenAPI契约）

### 核心特性
- ✅ **OpenAPI 3.1**：统一API契约，自动生成文档与客户端
- ✅ **ActivityStreams 2.0**：标准化活动事件模型，完整审计追踪
- ✅ **ULID + RFC3339**：时间排序一致性保证
- ✅ **幂等操作**：基于Idempotency-Key的可靠投递
- ✅ **类型安全**：端到端TypeScript类型推导
- ✅ **响应式设计**：移动端友好的Dashboard布局

---

## 🚀 快速开始

### 前置要求
- Node.js 20+ 
- PostgreSQL 15+
- Docker & Docker Compose
- Git

### 开发环境搭建（5分钟）

```bash
# 1. 克隆项目（假设已有仓库）
git clone <your-repo-url>
cd teamos

# 2. 安装依赖
npm install

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 配置数据库连接

# 4. 初始化数据库
npx prisma migrate dev

# 5. 启动开发服务器
npm run dev:backend  # 后端: http://localhost:3000
npm run dev:frontend # 前端: http://localhost:3001
```

### 验证安装
- 访问 `http://localhost:3000/api` 查看Swagger文档
- 访问 `http://localhost:3001` 查看前端Dashboard
- 访问 `http://localhost:3000/api/activities` 查看ActivityStreams端点

---

## 📖 学习路径

### 新手入门（建议顺序）
1. **阅读技术选型**：理解为什么选择这些技术栈
2. **搭建开发环境**：按照集成部署文档配置本地环境
3. **理解OpenAPI契约**：学习API设计规范与工具链
4. **实现第一个模块**：跟随后端/前端指南实现PRD模块
5. **集成ActivityStreams**：理解活动事件模型与审计追踪
6. **接入AI Agent**：配置Agent角色与协作流程

### 进阶开发
- **扩展新功能**：基于现有模式添加新的资源类型
- **优化性能**：缓存策略、数据库索引、查询优化
- **增强安全**：RBAC权限、审计日志、安全扫描
- **自定义Agent**：开发特定领域的AI Agent工具

---

## 🔧 开发规范

### 代码风格
- **TypeScript**：严格模式，禁用`any`
- **ESLint + Prettier**：自动格式化
- **Commit规范**：遵循Conventional Commits

### API设计原则
- **RESTful**：资源导向，HTTP动词语义化
- **OpenAPI优先**：先定义契约，再实现代码
- **版本化**：URL路径版本（/api/v1/...）
- **错误处理**：RFC7807 Problem Details

### 数据库规范
- **命名**：snake_case表名，camelCase字段名（Prisma映射）
- **迁移**：每次Schema变更必须生成迁移文件
- **索引**：查询字段必须建立索引
- **外键**：显式定义关系约束

---

## 🤝 AI Agent协作模式

### Agent角色定义
- **PM Agent**：需求分析、PRD编写、优先级排序
- **Design Agent**：UI/UX设计、设计稿生成、设计系统维护
- **Frontend Agent**：前端代码实现、组件开发、样式调整
- **Backend Agent**：API实现、数据库设计、业务逻辑
- **QA Agent**：测试用例生成、自动化测试、Bug报告
- **Build Agent**：CI/CD配置、构建优化、部署管理
- **Security Agent**：安全扫描、漏洞修复、权限审计

### 协作流程
```
需求输入 → PM Agent (生成PRD) 
         → Design Agent (生成设计稿)
         → Frontend/Backend Agent (并行开发)
         → QA Agent (测试验证)
         → Build Agent (构建部署)
         → Security Agent (安全审计)
```

所有活动通过ActivityStreams 2.0记录，形成完整的可追溯链路。

---

## 📊 项目里程碑

### Phase 1: 基础设施（2-3周）
- [x] 技术选型与架构设计
- [ ] 开发环境搭建
- [ ] OpenAPI契约定义
- [ ] 数据库Schema设计
- [ ] CI/CD流水线配置

### Phase 2: 核心功能（4-6周）
- [ ] PRD管理模块
- [ ] 设计稿管理模块
- [ ] Bug追踪模块
- [ ] 构建状态模块
- [ ] ActivityStreams集成

### Phase 3: AI Agent集成（3-4周）
- [ ] Agent接口规范
- [ ] PM/Design/Dev Agent实现
- [ ] QA/Build/Security Agent实现
- [ ] Agent协作流程测试

### Phase 4: 优化与上线（2-3周）
- [ ] 性能优化
- [ ] 安全加固
- [ ] 文档完善
- [ ] 生产部署

---

## 🐛 故障排查

### 常见问题
- **数据库连接失败**：检查`.env`配置与PostgreSQL服务状态
- **Prisma迁移错误**：清空数据库后重新`migrate dev`
- **OpenAPI类型不匹配**：重新生成客户端`npm run generate:api`
- **Agent调用超时**：检查网络连接与API密钥配置

### 调试技巧
- 使用`DEBUG=*`环境变量启用详细日志
- Swagger UI测试API端点
- Prisma Studio可视化数据库
- Chrome DevTools分析前端性能

---

## 📚 参考资源

### 官方文档
- [Next.js Documentation](https://nextjs.org/docs)
- [NestJS Documentation](https://docs.nestjs.com)
- [Prisma Documentation](https://www.prisma.io/docs)
- [OpenAPI Specification](https://spec.openapis.org/oas/v3.1.0)
- [ActivityStreams 2.0](https://www.w3.org/TR/activitystreams-core/)

### 社区资源
- [TanStack Query](https://tanstack.com/query)
- [shadcn/ui](https://ui.shadcn.com)
- [Spectral](https://stoplight.io/open-source/spectral)
- [Redocly](https://redocly.com/docs)

---

## 🤝 贡献指南

本项目欢迎贡献！请遵循以下流程：
1. Fork项目并创建特性分支
2. 遵循代码规范与Commit规范
3. 添加必要的测试用例
4. 提交Pull Request并描述变更内容

---

## 📄 许可证

本项目采用 MIT 许可证。详见 [LICENSE](../LICENSE) 文件。

---

## 📮 联系方式

- **问题反馈**：提交GitHub Issue
- **功能建议**：提交GitHub Discussion
- **安全漏洞**：发送邮件至 security@example.com

---

**开始构建你的个人级工程操作系统吧！** 🚀


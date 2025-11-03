# OpenAPI 工具链指南

> **Spectral校验、Redocly文档、客户端生成、版本化策略**

---

## 📋 目录
- [OpenAPI规范管理](#openapi规范管理)
- [Spectral规则校验](#spectral规则校验)
- [Redocly文档生成](#redocly文档生成)
- [客户端代码生成](#客户端代码生成)
- [API版本化策略](#api版本化策略)
- [完整工作流](#完整工作流)

---

## OpenAPI规范管理

### 1. 规范文件组织
```
api-specs/
├── openapi.yaml              # 主规范文件
├── components/               # 可复用组件
│   ├── schemas/
│   │   ├── prd.yaml
│   │   ├── design.yaml
│   │   ├── bug.yaml
│   │   └── common.yaml
│   ├── parameters/
│   │   └── pagination.yaml
│   ├── responses/
│   │   └── errors.yaml
│   └── securitySchemes/
│       └── bearer.yaml
├── paths/                    # API路径定义
│   ├── prds.yaml
│   ├── designs.yaml
│   ├── bugs.yaml
│   └── activities.yaml
└── .spectral.yaml           # Spectral配置
```

### 2. 主规范文件（openapi.yaml）
```yaml
openapi: 3.1.0
info:
  title: TeamOS API
  version: 1.0.0
  description: |
    个人级工程操作系统API文档
    
    ## 认证
    使用Bearer Token认证，在请求头中添加：
    ```
    Authorization: Bearer <your-token>
    ```
    
    ## 幂等性
    支持Idempotency-Key头，确保重复请求的幂等性：
    ```
    Idempotency-Key: <uuid>
    ```
  contact:
    name: TeamOS Support
    email: support@teamos.example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: http://localhost:3000
    description: 本地开发环境
  - url: https://api-staging.teamos.example.com
    description: 测试环境
  - url: https://api.teamos.example.com
    description: 生产环境

tags:
  - name: PRD
    description: PRD管理
  - name: Design
    description: 设计稿管理
  - name: Bug
    description: Bug追踪
  - name: Activity
    description: 活动流
  - name: Auth
    description: 认证授权

paths:
  # 引用外部路径定义
  /api/v1/prds:
    $ref: './paths/prds.yaml#/prds'
  /api/v1/prds/{id}:
    $ref: './paths/prds.yaml#/prd-by-id'
  /api/v1/designs:
    $ref: './paths/designs.yaml#/designs'
  /api/v1/bugs:
    $ref: './paths/bugs.yaml#/bugs'
  /api/v1/activities:
    $ref: './paths/activities.yaml#/activities'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

### 3. Schema定义示例（components/schemas/prd.yaml）
```yaml
PRDResponseDto:
  type: object
  required:
    - id
    - title
    - content
    - status
    - priority
    - authorId
    - createdAt
    - updatedAt
  properties:
    id:
      type: string
      format: cuid
      example: clxyz123abc
    title:
      type: string
      minLength: 1
      maxLength: 200
      example: 用户登录功能需求
    summary:
      type: string
      nullable: true
      example: 实现用户邮箱/密码登录功能
    content:
      type: string
      format: markdown
      example: |
        ## 背景
        用户需要登录系统...
    status:
      $ref: '#/PRDStatus'
    priority:
      $ref: '#/Priority'
    authorId:
      type: string
      format: cuid
    createdAt:
      type: string
      format: date-time
      example: '2024-01-15T10:30:00Z'
    updatedAt:
      type: string
      format: date-time
      example: '2024-01-15T10:30:00Z'

PRDStatus:
  type: string
  enum:
    - DRAFT
    - REVIEW
    - APPROVED
    - REJECTED
    - ARCHIVED

Priority:
  type: string
  enum:
    - LOW
    - MEDIUM
    - HIGH
    - URGENT

CreatePRDDto:
  type: object
  required:
    - title
    - content
  properties:
    title:
      type: string
      minLength: 1
      maxLength: 200
    summary:
      type: string
    content:
      type: string
    priority:
      $ref: '#/Priority'
      default: MEDIUM
```

---

## Spectral规则校验

### 1. 安装Spectral
```bash
npm install -g @stoplight/spectral-cli
```

### 2. 配置文件（.spectral.yaml）
```yaml
extends:
  - spectral:oas  # 使用官方OpenAPI规则集

rules:
  # 操作必须有operationId
  operation-operationId: error
  
  # 操作必须有标签
  operation-tags: error
  
  # 操作必须有摘要
  operation-summary: error
  
  # 操作必须有描述
  operation-description: warn
  
  # 路径参数必须有描述
  path-params: error
  
  # 响应必须有描述
  operation-success-response: error
  
  # 禁止模糊路径
  no-ambiguous-paths: error
  
  # Schema必须有描述
  oas3-schema-description: warn
  
  # 自定义规则：强制使用Bearer认证
  require-bearer-auth:
    description: API必须使用Bearer认证
    given: $.paths[*][*]
    severity: error
    then:
      - field: security
        function: truthy
  
  # 自定义规则：响应必须包含错误处理
  require-error-responses:
    description: 操作必须定义4xx和5xx错误响应
    given: $.paths[*][*].responses
    severity: error
    then:
      - field: '4[0-9]{2}'
        function: truthy
      - field: '5[0-9]{2}'
        function: truthy
  
  # 自定义规则：分页参数标准化
  pagination-parameters:
    description: 列表接口必须支持page和pageSize参数
    given: $.paths[*].get.parameters[?(@.name == 'page' || @.name == 'pageSize')]
    severity: warn
    then:
      - field: schema.type
        function: pattern
        functionOptions:
          match: integer
```

### 3. 运行校验
```bash
# 校验规范文件
spectral lint api-specs/openapi.yaml

# 输出格式化结果
spectral lint api-specs/openapi.yaml --format stylish

# 输出JSON格式（用于CI集成）
spectral lint api-specs/openapi.yaml --format json

# 自动修复（部分规则）
spectral lint api-specs/openapi.yaml --format pretty
```

### 4. CI集成（GitHub Actions）
```yaml
# .github/workflows/api-lint.yml
name: API Spec Linting

on:
  pull_request:
    paths:
      - 'api-specs/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install Spectral
        run: npm install -g @stoplight/spectral-cli
      
      - name: Lint OpenAPI Spec
        run: spectral lint api-specs/openapi.yaml --format junit > spectral-report.xml
      
      - name: Publish Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: spectral-report.xml
```

---

## Redocly文档生成

### 1. 安装Redocly CLI
```bash
npm install -g @redocly/cli
```

### 2. 配置文件（redocly.yaml）
```yaml
apis:
  teamos@v1:
    root: api-specs/openapi.yaml

lint:
  extends:
    - recommended
  rules:
    operation-operationId-unique: error
    operation-summary: error
    tag-description: warn
    no-unused-components: warn

theme:
  openapi:
    schemaExpansionLevel: 2
    jsonSampleExpandLevel: 2
    generateCodeSamples:
      languages:
        - lang: curl
        - lang: javascript
        - lang: python
        - lang: typescript
```

### 3. 生成文档
```bash
# 校验规范
redocly lint api-specs/openapi.yaml

# 生成静态HTML文档
redocly build-docs api-specs/openapi.yaml -o docs/api.html

# 启动文档预览服务器
redocly preview-docs api-specs/openapi.yaml

# Bundle规范文件（合并所有$ref）
redocly bundle api-specs/openapi.yaml -o dist/openapi.json

# 生成多格式
redocly bundle api-specs/openapi.yaml -o dist/openapi.yaml --format yaml
```

### 4. 自定义主题
```yaml
# redocly.yaml
theme:
  openapi:
    theme:
      colors:
        primary:
          main: '#3B82F6'
        success:
          main: '#10B981'
      typography:
        fontSize: '16px'
        fontFamily: 'Inter, sans-serif'
      sidebar:
        backgroundColor: '#F9FAFB'
      rightPanel:
        backgroundColor: '#1F2937'
```

---

## 客户端代码生成

### 1. TypeScript客户端（openapi-typescript）
```bash
# 安装
npm install -D openapi-typescript
npm install openapi-fetch

# 生成类型
npx openapi-typescript http://localhost:3000/api/openapi.json -o src/lib/api/types.ts

# 或从本地文件
npx openapi-typescript api-specs/openapi.yaml -o src/lib/api/types.ts
```

**使用示例**：
```typescript
import createClient from 'openapi-fetch';
import type { paths } from './types';

const client = createClient<paths>({ 
  baseUrl: 'http://localhost:3000' 
});

// 类型安全的API调用
const { data, error } = await client.GET('/api/v1/prds/{id}', {
  params: { path: { id: '123' } },
});

if (error) {
  console.error(error);
} else {
  console.log(data.title); // 完全类型推导
}
```

### 2. Python客户端（openapi-python-client）
```bash
# 安装
pip install openapi-python-client

# 生成客户端
openapi-python-client generate --url http://localhost:3000/api/openapi.json

# 或从本地文件
openapi-python-client generate --path api-specs/openapi.yaml
```

**使用示例**：
```python
from teamos_api_client import Client
from teamos_api_client.api.prd import create_prd
from teamos_api_client.models import CreatePRDDto

client = Client(base_url="http://localhost:3000", token="your-token")

# 类型安全的API调用
dto = CreatePRDDto(
    title="用户登录功能",
    content="## 背景\n...",
    priority="HIGH"
)

prd = create_prd.sync(client=client, json_body=dto)
print(f"PRD ID: {prd.id}")
```

### 3. 自动化生成脚本（package.json）
```json
{
  "scripts": {
    "generate:api-types": "openapi-typescript http://localhost:3000/api/openapi.json -o src/lib/api/types.ts",
    "generate:api-docs": "redocly build-docs api-specs/openapi.yaml -o docs/api.html",
    "generate:api-bundle": "redocly bundle api-specs/openapi.yaml -o dist/openapi.json",
    "generate:all": "npm run generate:api-types && npm run generate:api-docs && npm run generate:api-bundle"
  }
}
```

---

## API版本化策略

### 1. URL路径版本化（推荐）
```yaml
# openapi.yaml
paths:
  /api/v1/prds:
    get:
      summary: 获取PRD列表（v1）
  
  /api/v2/prds:
    get:
      summary: 获取PRD列表（v2，支持高级过滤）
      parameters:
        - name: tags
          in: query
          schema:
            type: array
            items:
              type: string
```

**NestJS实现**：
```typescript
// src/modules/prd/prd.controller.ts
@Controller('api/v1/prds')
export class PRDControllerV1 {
  // v1实现
}

@Controller('api/v2/prds')
export class PRDControllerV2 {
  // v2实现（新增功能）
}
```

### 2. 版本弃用策略
```yaml
paths:
  /api/v1/prds:
    get:
      deprecated: true
      summary: 获取PRD列表（已弃用，请使用v2）
      description: |
        ⚠️ 此API将在2024-12-31后停止支持
        请迁移至 /api/v2/prds
```

### 3. 版本兼容性矩阵
```markdown
| 版本 | 发布日期 | 弃用日期 | 停止支持日期 | 状态 |
|------|----------|----------|--------------|------|
| v1   | 2024-01  | 2024-06  | 2024-12      | 弃用 |
| v2   | 2024-06  | -        | -            | 当前 |
| v3   | 2024-12  | -        | -            | 计划 |
```

---

## 完整工作流

### 1. 开发流程
```bash
# 1. 设计API（先写规范）
vim api-specs/paths/prds.yaml

# 2. 校验规范
spectral lint api-specs/openapi.yaml

# 3. 生成文档预览
redocly preview-docs api-specs/openapi.yaml

# 4. 实现后端（基于规范）
# 在NestJS中实现Controller和Service

# 5. 生成前端类型
npm run generate:api-types

# 6. 实现前端（使用生成的类型）
# 在Next.js中调用API

# 7. 集成测试
npm run test:e2e

# 8. 生成最终文档
npm run generate:api-docs
```

### 2. CI/CD集成
```yaml
# .github/workflows/api-workflow.yml
name: API Workflow

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate OpenAPI Spec
        run: |
          npm install -g @stoplight/spectral-cli @redocly/cli
          spectral lint api-specs/openapi.yaml
          redocly lint api-specs/openapi.yaml
      
      - name: Generate Documentation
        run: redocly build-docs api-specs/openapi.yaml -o docs/api.html
      
      - name: Deploy Docs to GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./docs
```

### 3. 版本发布检查清单
- [ ] OpenAPI规范通过Spectral校验
- [ ] 所有端点有完整的文档和示例
- [ ] 生成的客户端代码无类型错误
- [ ] 集成测试覆盖所有新增端点
- [ ] 更新CHANGELOG.md
- [ ] 标记弃用的API（如有）
- [ ] 更新版本兼容性矩阵
- [ ] 部署文档到生产环境

---

## 最佳实践

### 1. 规范设计原则
- ✅ **契约优先**：先定义OpenAPI规范，再实现代码
- ✅ **一致性**：统一命名规范、错误格式、分页参数
- ✅ **可扩展性**：使用$ref复用组件，便于维护
- ✅ **文档完整**：每个端点都有描述、示例、错误码

### 2. 工具链集成
- ✅ **自动化**：CI/CD自动校验、生成、部署
- ✅ **类型安全**：自动生成客户端，避免手写API调用
- ✅ **版本管理**：Git管理规范文件，语义化版本号
- ✅ **监控告警**：规范变更触发通知

### 3. 团队协作
- ✅ **评审流程**：API变更必须经过Code Review
- ✅ **变更通知**：重大变更提前通知使用方
- ✅ **迁移指南**：提供版本升级文档
- ✅ **反馈渠道**：建立API使用问题反馈机制

---

**OpenAPI工具链指南已完成！至此，TeamOS完整实施指南全部完成！** 🎉

## 下一步行动

1. **阅读完整指南**：按顺序阅读所有文档
2. **搭建开发环境**：参考 [集成与部署](./03-integration-deploy.md)
3. **实现第一个模块**：从PRD模块开始
4. **集成AI Agent**：配置Agent协作流程
5. **持续优化**：根据实际使用反馈改进

**祝你构建成功！** 🚀


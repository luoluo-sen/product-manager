# ActivityStreams 2.0 实现指南

> **W3C标准活动流模型、扩展、存储与查询设计**

---

## 📋 目录
- [ActivityStreams 2.0 概述](#activitystreams-20-概述)
- [核心类型与模型](#核心类型与模型)
- [词汇扩展](#词汇扩展)
- [存储设计](#存储设计)
- [查询与分页](#查询与分页)
- [实现示例](#实现示例)

---

## ActivityStreams 2.0 概述

### 1. 什么是ActivityStreams 2.0？
ActivityStreams 2.0（AS2）是W3C推荐的标准，用于描述社交网络中的活动事件。它基于JSON-LD，提供了一套标准化的词汇表来表示：
- **谁（Actor）** 做了什么（Activity）
- **对什么（Object）** 进行了操作
- **何时（published）** 发生
- **在哪里（location）** 发生（可选）

### 2. 为什么选择AS2？
- ✅ **标准化**：W3C推荐规范，互操作性强
- ✅ **可扩展**：基于JSON-LD，支持自定义词汇
- ✅ **语义化**：明确的类型系统和关系
- ✅ **审计追踪**：天然支持完整的操作历史
- ✅ **生态成熟**：ActivityPub、Mastodon等广泛采用

### 3. 核心概念
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "actor": {
    "type": "Person",
    "id": "https://example.com/users/alice",
    "name": "Alice"
  },
  "object": {
    "type": "Note",
    "content": "Hello World!"
  },
  "published": "2024-01-15T10:30:00Z"
}
```

---

## 核心类型与模型

### 1. Actor类型（谁）
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Person",
  "id": "urn:user:alice",
  "name": "Alice Smith",
  "summary": "Product Manager",
  "icon": {
    "type": "Image",
    "url": "https://example.com/avatar.jpg"
  }
}
```

**标准Actor类型**：
- `Person` - 个人用户
- `Application` - 应用程序
- `Service` - 服务（AI Agent适用）
- `Organization` - 组织
- `Group` - 群组

### 2. Activity类型（做了什么）
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Create",
  "id": "urn:activity:01HXYZ...",
  "actor": "urn:user:alice",
  "object": {
    "type": "Note",
    "id": "urn:prd:123",
    "name": "用户登录功能PRD",
    "content": "## 背景\n..."
  },
  "published": "2024-01-15T10:30:00Z"
}
```

**常用Activity类型**：
- `Create` - 创建对象
- `Update` - 更新对象
- `Delete` - 删除对象
- `Add` - 添加到集合
- `Remove` - 从集合移除
- `Like` - 点赞
- `Announce` - 公告/转发
- `Follow` - 关注
- `Accept` - 接受
- `Reject` - 拒绝

### 3. Object类型（对什么）
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Note",
  "id": "urn:prd:123",
  "name": "用户登录功能PRD",
  "content": "## 背景\n用户需要登录系统...",
  "attributedTo": "urn:user:alice",
  "published": "2024-01-15T10:30:00Z",
  "updated": "2024-01-15T11:00:00Z"
}
```

**标准Object类型**：
- `Note` - 短文本（适用PRD摘要）
- `Article` - 长文本（适用完整PRD）
- `Document` - 文档（适用设计稿）
- `Image` - 图片
- `Video` - 视频
- `Event` - 事件

### 4. Collection类型（集合与分页）
```json
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "OrderedCollection",
  "id": "urn:activities:all",
  "totalItems": 150,
  "first": {
    "type": "OrderedCollectionPage",
    "id": "urn:activities:all?page=1",
    "orderedItems": [
      { "type": "Create", "...": "..." },
      { "type": "Update", "...": "..." }
    ],
    "next": "urn:activities:all?page=2"
  }
}
```

---

## 词汇扩展

### 1. 自定义Context
```json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    {
      "teamos": "https://teamos.example.com/ns#",
      "PRD": "teamos:PRD",
      "Design": "teamos:Design",
      "Bug": "teamos:Bug",
      "Build": "teamos:Build",
      "priority": "teamos:priority",
      "severity": "teamos:severity",
      "status": "teamos:status"
    }
  ],
  "type": "Create",
  "actor": "urn:user:alice",
  "object": {
    "type": "PRD",
    "id": "urn:prd:123",
    "name": "用户登录功能",
    "priority": "HIGH",
    "status": "DRAFT"
  }
}
```

### 2. 扩展属性映射
```typescript
// src/modules/activity/activity-mapper.ts
export class ActivityMapper {
  static toPRDActivity(prd: PRD, actorId: string, type: string) {
    return {
      '@context': [
        'https://www.w3.org/ns/activitystreams',
        {
          teamos: 'https://teamos.example.com/ns#',
          PRD: 'teamos:PRD',
          priority: 'teamos:priority',
          status: 'teamos:status',
        },
      ],
      type,
      id: `urn:activity:${Date.now()}`,
      actor: {
        type: 'Person',
        id: `urn:user:${actorId}`,
      },
      object: {
        type: 'PRD',
        id: `urn:prd:${prd.id}`,
        name: prd.title,
        summary: prd.summary,
        content: prd.content,
        priority: prd.priority,
        status: prd.status,
        attributedTo: `urn:user:${prd.authorId}`,
      },
      published: new Date().toISOString(),
    };
  }

  static toDesignActivity(design: Design, actorId: string, type: string) {
    return {
      '@context': [
        'https://www.w3.org/ns/activitystreams',
        {
          teamos: 'https://teamos.example.com/ns#',
          Design: 'teamos:Design',
        },
      ],
      type,
      actor: { type: 'Person', id: `urn:user:${actorId}` },
      object: {
        type: 'Design',
        id: `urn:design:${design.id}`,
        name: design.title,
        url: design.fileUrl,
        thumbnail: design.thumbnail,
        status: design.status,
      },
      published: new Date().toISOString(),
    };
  }

  static toBugActivity(bug: Bug, actorId: string, type: string) {
    return {
      '@context': [
        'https://www.w3.org/ns/activitystreams',
        {
          teamos: 'https://teamos.example.com/ns#',
          Bug: 'teamos:Bug',
          severity: 'teamos:severity',
        },
      ],
      type,
      actor: { type: 'Person', id: `urn:user:${actorId}` },
      object: {
        type: 'Bug',
        id: `urn:bug:${bug.id}`,
        name: bug.title,
        content: bug.description,
        severity: bug.severity,
        status: bug.status,
      },
      published: new Date().toISOString(),
    };
  }
}
```

---

## 存储设计

### 1. 混合存储策略
```prisma
model Activity {
  id        String   @id @default(cuid())
  
  // 结构化字段（用于快速查询）
  type      String   // Create, Update, Delete
  actorId   String?
  actor     User?    @relation(fields: [actorId], references: [id])
  
  // 对象引用（多态）
  objectType String? // PRD, Design, Bug
  objectId   String?
  
  // 关系（可选，用于JOIN查询）
  prdId      String?
  prd        PRD?     @relation(fields: [prdId], references: [id])
  designId   String?
  design     Design?  @relation(fields: [designId], references: [id])
  bugId      String?
  bug        Bug?     @relation(fields: [bugId], references: [id])
  
  // 完整AS2对象（JSONB存储）
  payload   Json     // 完整的ActivityStreams对象
  
  // 时间戳
  published DateTime @default(now())
  
  // 索引
  @@index([actorId])
  @@index([type])
  @@index([objectType, objectId])
  @@index([published])
  @@index([prdId])
  @@index([designId])
  @@index([bugId])
  @@map("activities")
}
```

### 2. 索引策略
```sql
-- 复合索引：按类型和时间查询
CREATE INDEX idx_activities_type_published 
ON activities(type, published DESC);

-- JSONB索引：查询payload中的特定字段
CREATE INDEX idx_activities_payload_type 
ON activities USING GIN ((payload->'type'));

CREATE INDEX idx_activities_payload_actor 
ON activities USING GIN ((payload->'actor'));

-- 全文搜索索引
CREATE INDEX idx_activities_payload_fulltext 
ON activities USING GIN (to_tsvector('english', payload::text));
```

---

## 查询与分页

### 1. 基础查询（NestJS Service）
```typescript
// src/modules/activity/activity.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class ActivityService {
  constructor(private prisma: PrismaService) {}

  async getActivities(params: {
    page?: number;
    pageSize?: number;
    type?: string;
    actorId?: string;
    objectType?: string;
    startDate?: string;
    endDate?: string;
  }) {
    const {
      page = 1,
      pageSize = 20,
      type,
      actorId,
      objectType,
      startDate,
      endDate,
    } = params;

    const where: any = {};
    if (type) where.type = type;
    if (actorId) where.actorId = actorId;
    if (objectType) where.objectType = objectType;
    if (startDate || endDate) {
      where.published = {};
      if (startDate) where.published.gte = new Date(startDate);
      if (endDate) where.published.lte = new Date(endDate);
    }

    const skip = (page - 1) * pageSize;

    const [items, total] = await Promise.all([
      this.prisma.activity.findMany({
        where,
        skip,
        take: pageSize,
        orderBy: { published: 'desc' },
        include: {
          actor: { select: { id: true, name: true, email: true } },
        },
      }),
      this.prisma.activity.count({ where }),
    ]);

    return {
      '@context': 'https://www.w3.org/ns/activitystreams',
      type: 'OrderedCollection',
      totalItems: total,
      items: items.map((item) => item.payload),
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize),
    };
  }

  async getActivityById(id: string) {
    const activity = await this.prisma.activity.findUnique({
      where: { id },
      include: { actor: true },
    });

    if (!activity) {
      throw new NotFoundException(`Activity ${id} not found`);
    }

    return activity.payload;
  }

  async getActivitiesByObject(objectType: string, objectId: string) {
    const activities = await this.prisma.activity.findMany({
      where: { objectType, objectId },
      orderBy: { published: 'desc' },
      include: { actor: true },
    });

    return {
      '@context': 'https://www.w3.org/ns/activitystreams',
      type: 'OrderedCollection',
      totalItems: activities.length,
      items: activities.map((a) => a.payload),
    };
  }
}
```

### 2. 分页响应格式
```typescript
// src/modules/activity/activity.controller.ts
@Get()
@ApiOperation({ summary: '获取活动流' })
@ApiQuery({ name: 'page', required: false, type: Number })
@ApiQuery({ name: 'pageSize', required: false, type: Number })
async getActivities(
  @Query('page') page = 1,
  @Query('pageSize') pageSize = 20,
  @Query('type') type?: string,
  @Query('actorId') actorId?: string,
) {
  const result = await this.activityService.getActivities({
    page,
    pageSize,
    type,
    actorId,
  });

  // 添加分页链接
  const baseUrl = 'https://api.teamos.example.com/api/v1/activities';
  const response = {
    ...result,
    first: `${baseUrl}?page=1&pageSize=${pageSize}`,
    last: `${baseUrl}?page=${result.totalPages}&pageSize=${pageSize}`,
  };

  if (page > 1) {
    response['prev'] = `${baseUrl}?page=${page - 1}&pageSize=${pageSize}`;
  }
  if (page < result.totalPages) {
    response['next'] = `${baseUrl}?page=${page + 1}&pageSize=${pageSize}`;
  }

  return response;
}
```

---

## 实现示例

### 1. 创建Activity（完整流程）
```typescript
// src/modules/prd/prd.service.ts
async create(dto: CreatePRDDto, authorId: string) {
  // 1. 创建PRD
  const prd = await this.prisma.pRD.create({
    data: { ...dto, authorId },
  });

  // 2. 生成ActivityStreams对象
  const activityPayload = ActivityMapper.toPRDActivity(
    prd,
    authorId,
    'Create',
  );

  // 3. 存储Activity
  await this.prisma.activity.create({
    data: {
      type: 'Create',
      actorId: authorId,
      objectType: 'PRD',
      objectId: prd.id,
      prdId: prd.id,
      payload: activityPayload,
    },
  });

  return prd;
}
```

### 2. 查询对象的完整历史
```typescript
async getPRDHistory(prdId: string) {
  const activities = await this.prisma.activity.findMany({
    where: {
      objectType: 'PRD',
      objectId: prdId,
    },
    orderBy: { published: 'asc' },
    include: { actor: true },
  });

  return {
    '@context': 'https://www.w3.org/ns/activitystreams',
    type: 'OrderedCollection',
    summary: `History of PRD ${prdId}`,
    totalItems: activities.length,
    orderedItems: activities.map((a) => a.payload),
  };
}
```

### 3. 前端展示Activity Feed
```typescript
// src/components/activity/activity-feed.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { format } from 'date-fns';
import { zhCN } from 'date-fns/locale';

export function ActivityFeed({ objectType, objectId }: { 
  objectType?: string; 
  objectId?: string 
}) {
  const { data, isLoading } = useQuery({
    queryKey: ['activities', objectType, objectId],
    queryFn: async () => {
      const params = new URLSearchParams();
      if (objectType) params.set('objectType', objectType);
      if (objectId) params.set('objectId', objectId);
      
      const response = await fetch(
        `/api/v1/activities?${params.toString()}`
      );
      return response.json();
    },
  });

  if (isLoading) return <div>加载中...</div>;

  return (
    <div className="space-y-4">
      {data?.items?.map((activity: any) => (
        <div key={activity.id} className="rounded-lg border p-4">
          <div className="flex items-start gap-3">
            <div className="h-10 w-10 rounded-full bg-primary/10" />
            <div className="flex-1">
              <p className="font-medium">
                {activity.actor.name}
                <span className="ml-2 text-sm text-muted-foreground">
                  {getActivityText(activity.type)}
                </span>
              </p>
              <p className="text-sm text-muted-foreground">
                {activity.object.name}
              </p>
              <p className="mt-1 text-xs text-muted-foreground">
                {format(new Date(activity.published), 'PPpp', { locale: zhCN })}
              </p>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
}

function getActivityText(type: string): string {
  const map: Record<string, string> = {
    Create: '创建了',
    Update: '更新了',
    Delete: '删除了',
    Add: '添加了',
    Remove: '移除了',
  };
  return map[type] || type;
}
```

---

## 最佳实践

### 1. 性能优化
- ✅ 使用JSONB索引加速查询
- ✅ 分页查询避免全表扫描
- ✅ 缓存热点Activity（Redis）
- ✅ 异步写入Activity（消息队列）

### 2. 数据一致性
- ✅ 使用事务保证PRD和Activity同时创建
- ✅ 软删除：Delete Activity不删除原对象
- ✅ 版本控制：Update Activity保留历史版本

### 3. 隐私与权限
- ✅ 根据用户权限过滤Activity
- ✅ 敏感信息脱敏（如密码、Token）
- ✅ 支持私有Activity（仅特定用户可见）

---

**ActivityStreams 2.0实现已完成！继续阅读 [OpenAPI工具链](./06-openapi-tooling.md)** 📊


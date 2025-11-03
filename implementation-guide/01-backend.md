# 后端实现指南

> **NestJS + Prisma + OpenAPI 完整实现步骤**

---

## 📋 目录
- [环境准备](#环境准备)
- [项目初始化](#项目初始化)
- [数据库设计](#数据库设计)
- [OpenAPI规范定义](#openapi规范定义)
- [模块化架构](#模块化架构)
- [认证授权](#认证授权)
- [ActivityStreams集成](#activitystreams集成)
- [测试策略](#测试策略)

---

## 环境准备

### 前置要求
```bash
node -v    # v20.0.0+
npm -v     # v10.0.0+
psql --version  # PostgreSQL 15+
```

### 安装NestJS CLI
```bash
npm install -g @nestjs/cli
nest --version  # 验证安装
```

---

## 项目初始化

### 1. 创建NestJS项目
```bash
nest new teamos-backend
cd teamos-backend

# 选择包管理器: npm
# 项目结构:
# src/
# ├── app.controller.ts
# ├── app.module.ts
# ├── app.service.ts
# └── main.ts
```

### 2. 安装核心依赖
```bash
# Prisma ORM
npm install @prisma/client
npm install -D prisma

# OpenAPI/Swagger
npm install @nestjs/swagger

# 验证与转换
npm install class-validator class-transformer

# 配置管理
npm install @nestjs/config

# JWT认证
npm install @nestjs/jwt @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

### 3. 初始化Prisma
```bash
npx prisma init

# 生成文件:
# prisma/
# ├── schema.prisma
# └── .env
```

### 4. 配置环境变量
```bash
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/teamos?schema=public"
JWT_SECRET="your-super-secret-key-change-in-production"
JWT_EXPIRES_IN="7d"
NODE_ENV="development"
PORT=3000
```

---

## 数据库设计

### Prisma Schema 设计

#### 1. 基础模型（prisma/schema.prisma）
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 用户模型
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  password  String   // bcrypt hash
  role      Role     @default(USER)
  
  // 关系
  prds      PRD[]
  designs   Design[]
  bugs      Bug[]
  activities Activity[]
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

enum Role {
  USER
  ADMIN
  AGENT
}

// PRD模型
model PRD {
  id          String      @id @default(cuid())
  title       String
  summary     String?
  content     String      @db.Text
  status      PRDStatus   @default(DRAFT)
  priority    Priority    @default(MEDIUM)
  
  // 关系
  authorId    String
  author      User        @relation(fields: [authorId], references: [id])
  activities  Activity[]
  
  // 元数据
  metadata    Json?       // 扩展字段
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  @@index([authorId])
  @@index([status])
  @@index([priority])
  @@map("prds")
}

enum PRDStatus {
  DRAFT
  REVIEW
  APPROVED
  REJECTED
  ARCHIVED
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

// 设计稿模型
model Design {
  id          String       @id @default(cuid())
  title       String
  description String?
  fileUrl     String       // 设计文件URL
  thumbnail   String?      // 缩略图URL
  status      DesignStatus @default(DRAFT)
  
  // 关系
  designerId  String
  designer    User         @relation(fields: [designerId], references: [id])
  prdId       String?      // 可选关联PRD
  activities  Activity[]
  
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt

  @@index([designerId])
  @@index([status])
  @@map("designs")
}

enum DesignStatus {
  DRAFT
  REVIEW
  APPROVED
  IMPLEMENTED
}

// Bug模型
model Bug {
  id          String    @id @default(cuid())
  title       String
  description String    @db.Text
  severity    Severity  @default(MEDIUM)
  status      BugStatus @default(OPEN)
  
  // 关系
  reporterId  String
  reporter    User      @relation(fields: [reporterId], references: [id])
  assigneeId  String?
  activities  Activity[]
  
  // 元数据
  stackTrace  String?   @db.Text
  environment String?   // dev/staging/production
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  resolvedAt  DateTime?

  @@index([reporterId])
  @@index([status])
  @@index([severity])
  @@map("bugs")
}

enum Severity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

enum BugStatus {
  OPEN
  IN_PROGRESS
  RESOLVED
  CLOSED
  WONT_FIX
}

// ActivityStreams 2.0 活动模型
model Activity {
  id        String   @id @default(cuid())
  
  // AS2.0 核心字段
  type      String   // Create, Update, Delete, etc.
  actorId   String?
  actor     User?    @relation(fields: [actorId], references: [id])
  
  // 对象引用（多态）
  objectType String? // PRD, Design, Bug, etc.
  objectId   String?
  
  // 关系（可选，用于快速查询）
  prdId      String?
  prd        PRD?     @relation(fields: [prdId], references: [id])
  designId   String?
  design     Design?  @relation(fields: [designId], references: [id])
  bugId      String?
  bug        Bug?     @relation(fields: [bugId], references: [id])
  
  // AS2.0 完整对象（JSONB存储）
  payload   Json     // 完整的ActivityStreams对象
  
  // 元数据
  published DateTime @default(now())
  
  @@index([actorId])
  @@index([type])
  @@index([objectType, objectId])
  @@index([published])
  @@map("activities")
}
```

#### 2. 生成迁移
```bash
# 创建初始迁移
npx prisma migrate dev --name init

# 生成Prisma Client
npx prisma generate

# 查看数据库（可选）
npx prisma studio
```

---

## OpenAPI规范定义

### 1. 配置Swagger模块（src/main.ts）
```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // 全局验证管道
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,           // 移除未定义的属性
    forbidNonWhitelisted: true, // 拒绝未定义的属性
    transform: true,            // 自动类型转换
  }));
  
  // CORS配置
  app.enableCors({
    origin: process.env.FRONTEND_URL || 'http://localhost:3001',
    credentials: true,
  });
  
  // Swagger配置
  const config = new DocumentBuilder()
    .setTitle('TeamOS API')
    .setDescription('个人级工程操作系统API文档')
    .setVersion('1.0')
    .addBearerAuth()
    .addTag('PRD', 'PRD管理')
    .addTag('Design', '设计稿管理')
    .addTag('Bug', 'Bug追踪')
    .addTag('Activity', '活动流')
    .addTag('Auth', '认证授权')
    .build();
  
  const documentFactory = () => SwaggerModule.createDocument(app, config, {
    operationIdFactory: (controllerKey: string, methodKey: string) => methodKey,
  });
  
  SwaggerModule.setup('api', app, documentFactory, {
    jsonDocumentUrl: 'api/openapi.json',
    yamlDocumentUrl: 'api/openapi.yaml',
  });
  
  const port = process.env.PORT || 3000;
  await app.listen(port);
  console.log(`🚀 Server running on http://localhost:${port}`);
  console.log(`📚 API Docs: http://localhost:${port}/api`);
}

bootstrap();
```

### 2. DTO定义示例（src/modules/prd/dto/）

#### create-prd.dto.ts
```typescript
import { ApiProperty } from '@nestjs/swagger';
import { IsString, IsEnum, IsOptional, MaxLength, MinLength } from 'class-validator';
import { PRDStatus, Priority } from '@prisma/client';

export class CreatePRDDto {
  @ApiProperty({
    description: 'PRD标题',
    example: '用户登录功能需求',
    minLength: 1,
    maxLength: 200,
  })
  @IsString()
  @MinLength(1)
  @MaxLength(200)
  title: string;

  @ApiProperty({
    description: 'PRD摘要',
    example: '实现用户邮箱/密码登录功能',
    required: false,
  })
  @IsString()
  @IsOptional()
  summary?: string;

  @ApiProperty({
    description: 'PRD详细内容（Markdown格式）',
    example: '## 背景\n用户需要登录系统...',
  })
  @IsString()
  content: string;

  @ApiProperty({
    description: '优先级',
    enum: Priority,
    default: Priority.MEDIUM,
  })
  @IsEnum(Priority)
  @IsOptional()
  priority?: Priority;
}
```

#### prd-response.dto.ts
```typescript
import { ApiProperty } from '@nestjs/swagger';
import { PRDStatus, Priority } from '@prisma/client';

export class PRDResponseDto {
  @ApiProperty({ example: 'clxyz123...' })
  id: string;

  @ApiProperty({ example: '用户登录功能需求' })
  title: string;

  @ApiProperty({ example: '实现用户邮箱/密码登录功能' })
  summary: string | null;

  @ApiProperty({ example: '## 背景\n...' })
  content: string;

  @ApiProperty({ enum: PRDStatus, example: PRDStatus.DRAFT })
  status: PRDStatus;

  @ApiProperty({ enum: Priority, example: Priority.MEDIUM })
  priority: Priority;

  @ApiProperty({ example: 'cluser123...' })
  authorId: string;

  @ApiProperty({ example: '2024-01-15T10:30:00Z' })
  createdAt: Date;

  @ApiProperty({ example: '2024-01-15T10:30:00Z' })
  updatedAt: Date;
}
```

---

## 模块化架构

### 1. PRD模块完整实现

#### prd.module.ts
```typescript
import { Module } from '@nestjs/common';
import { PRDController } from './prd.controller';
import { PRDService } from './prd.service';
import { PrismaModule } from '../prisma/prisma.module';
import { ActivityModule } from '../activity/activity.module';

@Module({
  imports: [PrismaModule, ActivityModule],
  controllers: [PRDController],
  providers: [PRDService],
  exports: [PRDService],
})
export class PRDModule {}
```

#### prd.controller.ts
```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  UseGuards,
  Request,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiQuery,
} from '@nestjs/swagger';
import { PRDService } from './prd.service';
import { CreatePRDDto } from './dto/create-prd.dto';
import { UpdatePRDDto } from './dto/update-prd.dto';
import { PRDResponseDto } from './dto/prd-response.dto';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { PRDStatus } from '@prisma/client';

@ApiTags('PRD')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('api/v1/prds')
export class PRDController {
  constructor(private readonly prdService: PRDService) {}

  @Post()
  @ApiOperation({ summary: '创建PRD' })
  @ApiResponse({ status: 201, type: PRDResponseDto })
  async create(@Body() dto: CreatePRDDto, @Request() req) {
    return this.prdService.create(dto, req.user.id);
  }

  @Get()
  @ApiOperation({ summary: '获取PRD列表' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'pageSize', required: false, type: Number })
  @ApiQuery({ name: 'status', required: false, enum: PRDStatus })
  @ApiResponse({ status: 200, type: [PRDResponseDto] })
  async findAll(
    @Query('page') page = 1,
    @Query('pageSize') pageSize = 20,
    @Query('status') status?: PRDStatus,
  ) {
    return this.prdService.findAll({ page, pageSize, status });
  }

  @Get(':id')
  @ApiOperation({ summary: '获取PRD详情' })
  @ApiResponse({ status: 200, type: PRDResponseDto })
  async findOne(@Param('id') id: string) {
    return this.prdService.findOne(id);
  }

  @Put(':id')
  @ApiOperation({ summary: '更新PRD' })
  @ApiResponse({ status: 200, type: PRDResponseDto })
  async update(
    @Param('id') id: string,
    @Body() dto: UpdatePRDDto,
    @Request() req,
  ) {
    return this.prdService.update(id, dto, req.user.id);
  }

  @Delete(':id')
  @ApiOperation({ summary: '删除PRD' })
  @ApiResponse({ status: 204 })
  async remove(@Param('id') id: string, @Request() req) {
    return this.prdService.remove(id, req.user.id);
  }
}
```

#### prd.service.ts（部分）
```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { ActivityService } from '../activity/activity.service';
import { CreatePRDDto } from './dto/create-prd.dto';
import { UpdatePRDDto } from './dto/update-prd.dto';

@Injectable()
export class PRDService {
  constructor(
    private prisma: PrismaService,
    private activityService: ActivityService,
  ) {}

  async create(dto: CreatePRDDto, authorId: string) {
    const prd = await this.prisma.pRD.create({
      data: {
        ...dto,
        authorId,
      },
    });

    // 记录ActivityStreams事件
    await this.activityService.createActivity({
      type: 'Create',
      actorId: authorId,
      objectType: 'PRD',
      objectId: prd.id,
      object: prd,
    });

    return prd;
  }

  async findAll({ page, pageSize, status }) {
    const skip = (page - 1) * pageSize;
    const where = status ? { status } : {};

    const [items, total] = await Promise.all([
      this.prisma.pRD.findMany({
        where,
        skip,
        take: pageSize,
        orderBy: { createdAt: 'desc' },
        include: { author: { select: { id: true, name: true, email: true } } },
      }),
      this.prisma.pRD.count({ where }),
    ]);

    return {
      items,
      page,
      pageSize,
      total,
      totalPages: Math.ceil(total / pageSize),
    };
  }

  async findOne(id: string) {
    const prd = await this.prisma.pRD.findUnique({
      where: { id },
      include: {
        author: { select: { id: true, name: true, email: true } },
        activities: {
          take: 10,
          orderBy: { published: 'desc' },
        },
      },
    });

    if (!prd) {
      throw new NotFoundException(`PRD with ID ${id} not found`);
    }

    return prd;
  }

  // update 和 remove 方法类似，都需记录Activity
}
```

---

## 认证授权

### 1. JWT策略（src/modules/auth/strategies/jwt.strategy.ts）
```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PrismaService } from '../../prisma/prisma.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private prisma: PrismaService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET,
    });
  }

  async validate(payload: any) {
    const user = await this.prisma.user.findUnique({
      where: { id: payload.sub },
      select: { id: true, email: true, name: true, role: true },
    });

    if (!user) {
      throw new UnauthorizedException();
    }

    return user;
  }
}
```

### 2. Auth Guard（src/modules/auth/guards/jwt-auth.guard.ts）
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

### 3. 登录/注册（src/modules/auth/auth.service.ts）
```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { PrismaService } from '../prisma/prisma.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async register(email: string, password: string, name: string) {
    const hashedPassword = await bcrypt.hash(password, 10);
    
    const user = await this.prisma.user.create({
      data: {
        email,
        password: hashedPassword,
        name,
      },
    });

    return this.generateToken(user);
  }

  async login(email: string, password: string) {
    const user = await this.prisma.user.findUnique({ where: { email } });
    
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new UnauthorizedException('Invalid credentials');
    }

    return this.generateToken(user);
  }

  private generateToken(user: any) {
    const payload = { sub: user.id, email: user.email, role: user.role };
    return {
      access_token: this.jwtService.sign(payload),
      user: { id: user.id, email: user.email, name: user.name },
    };
  }
}
```

---

## ActivityStreams集成

### activity.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class ActivityService {
  constructor(private prisma: PrismaService) {}

  async createActivity(data: {
    type: string;
    actorId: string;
    objectType: string;
    objectId: string;
    object: any;
  }) {
    const payload = {
      '@context': 'https://www.w3.org/ns/activitystreams',
      type: data.type,
      id: `urn:activity:${Date.now()}`,
      actor: {
        type: 'Person',
        id: `urn:user:${data.actorId}`,
      },
      object: {
        type: data.objectType,
        id: `urn:${data.objectType.toLowerCase()}:${data.objectId}`,
        ...data.object,
      },
      published: new Date().toISOString(),
    };

    return this.prisma.activity.create({
      data: {
        type: data.type,
        actorId: data.actorId,
        objectType: data.objectType,
        objectId: data.objectId,
        payload,
      },
    });
  }

  async getActivities({ page = 1, pageSize = 20, type, actorId }) {
    const where: any = {};
    if (type) where.type = type;
    if (actorId) where.actorId = actorId;

    const skip = (page - 1) * pageSize;

    const [items, total] = await Promise.all([
      this.prisma.activity.findMany({
        where,
        skip,
        take: pageSize,
        orderBy: { published: 'desc' },
        include: { actor: { select: { id: true, name: true } } },
      }),
      this.prisma.activity.count({ where }),
    ]);

    return { items, page, pageSize, total };
  }
}
```

---

## 测试策略

### 单元测试示例（prd.service.spec.ts）
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { PRDService } from './prd.service';
import { PrismaService } from '../prisma/prisma.service';
import { ActivityService } from '../activity/activity.service';

describe('PRDService', () => {
  let service: PRDService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PRDService,
        {
          provide: PrismaService,
          useValue: {
            pRD: {
              create: jest.fn(),
              findMany: jest.fn(),
              findUnique: jest.fn(),
            },
          },
        },
        {
          provide: ActivityService,
          useValue: {
            createActivity: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<PRDService>(PRDService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  it('should create a PRD', async () => {
    const dto = { title: 'Test', content: 'Content' };
    const expected = { id: '1', ...dto, authorId: 'user1' };

    jest.spyOn(prisma.pRD, 'create').mockResolvedValue(expected as any);

    const result = await service.create(dto, 'user1');
    expect(result).toEqual(expected);
  });
});
```

---

**后端基础架构已完成！继续阅读 [前端实现指南](./02-frontend.md)** 🚀


# 前端实现指南

> **Next.js 15 + TanStack Query + shadcn/ui 完整实现**

---

## 📋 目录
- [项目初始化](#项目初始化)
- [项目结构](#项目结构)
- [OpenAPI客户端生成](#openapi客户端生成)
- [TanStack Query配置](#tanstack-query配置)
- [UI组件库集成](#ui组件库集成)
- [Dashboard布局设计](#dashboard布局设计)
- [核心页面实现](#核心页面实现)
- [响应式设计](#响应式设计)

---

## 项目初始化

### 1. 创建Next.js项目
```bash
npx create-next-app@latest teamos-frontend
# ✔ Would you like to use TypeScript? Yes
# ✔ Would you like to use ESLint? Yes
# ✔ Would you like to use Tailwind CSS? Yes
# ✔ Would you like to use `src/` directory? Yes
# ✔ Would you like to use App Router? Yes
# ✔ Would you like to customize the default import alias? No

cd teamos-frontend
```

### 2. 安装核心依赖
```bash
# TanStack Query
npm install @tanstack/react-query @tanstack/react-query-devtools

# OpenAPI客户端生成
npm install -D openapi-typescript
npm install openapi-fetch

# 表单处理
npm install react-hook-form @hookform/resolvers zod

# 日期处理
npm install date-fns

# Markdown渲染
npm install react-markdown remark-gfm
```

### 3. 配置环境变量
```bash
# .env.local
NEXT_PUBLIC_API_URL=http://localhost:3000
```

---

## 项目结构

```
src/
├── app/                      # App Router页面
│   ├── (auth)/              # 认证路由组
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/         # Dashboard路由组
│   │   ├── layout.tsx       # Dashboard布局
│   │   ├── page.tsx         # 首页
│   │   ├── prds/
│   │   │   ├── page.tsx     # PRD列表
│   │   │   ├── [id]/
│   │   │   │   └── page.tsx # PRD详情
│   │   │   └── new/
│   │   │       └── page.tsx # 创建PRD
│   │   ├── designs/
│   │   ├── bugs/
│   │   └── activities/
│   ├── layout.tsx           # 根布局
│   └── providers.tsx        # 全局Provider
├── components/
│   ├── ui/                  # shadcn/ui组件
│   ├── layout/              # 布局组件
│   │   ├── header.tsx
│   │   ├── sidebar.tsx
│   │   └── footer.tsx
│   ├── prd/                 # PRD相关组件
│   │   ├── prd-card.tsx
│   │   ├── prd-form.tsx
│   │   └── prd-list.tsx
│   └── activity/            # 活动流组件
│       └── activity-feed.tsx
├── lib/
│   ├── api/                 # API客户端
│   │   ├── client.ts        # OpenAPI客户端
│   │   └── types.ts         # 生成的类型
│   ├── hooks/               # 自定义Hooks
│   │   ├── use-prds.ts
│   │   ├── use-designs.ts
│   │   └── use-auth.ts
│   └── utils.ts             # 工具函数
└── types/                   # 类型定义
    └── index.ts
```

---

## OpenAPI客户端生成

### 1. 配置生成脚本（package.json）
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "generate:api": "openapi-typescript http://localhost:3000/api/openapi.json -o src/lib/api/types.ts"
  }
}
```

### 2. 创建API客户端（src/lib/api/client.ts）
```typescript
import createClient from 'openapi-fetch';
import type { paths } from './types';

const client = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000',
});

// 添加认证拦截器
client.use({
  async onRequest({ request }) {
    const token = localStorage.getItem('access_token');
    if (token) {
      request.headers.set('Authorization', `Bearer ${token}`);
    }
    return request;
  },
  async onResponse({ response }) {
    if (response.status === 401) {
      // 清除token并跳转登录
      localStorage.removeItem('access_token');
      window.location.href = '/login';
    }
    return response;
  },
});

export default client;
```

### 3. 生成类型
```bash
# 确保后端服务运行在 localhost:3000
npm run generate:api
```

---

## TanStack Query配置

### 1. 创建Provider（src/app/providers.tsx）
```typescript
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1分钟
            refetchOnWindowFocus: false,
          },
        },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### 2. 在根布局中使用（src/app/layout.tsx）
```typescript
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import { Providers } from './providers';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'TeamOS - 个人级工程操作系统',
  description: '将一个全栈工程师 + 多个AI Agent抽象成多团队协同模式',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="zh-CN">
      <body className={inter.className}>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### 3. 创建自定义Hooks（src/lib/hooks/use-prds.ts）
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import client from '@/lib/api/client';
import type { components } from '@/lib/api/types';

type PRD = components['schemas']['PRDResponseDto'];
type CreatePRDDto = components['schemas']['CreatePRDDto'];
type UpdatePRDDto = components['schemas']['UpdatePRDDto'];

// 查询PRD列表
export function usePRDs(params?: { page?: number; pageSize?: number; status?: string }) {
  return useQuery({
    queryKey: ['prds', params],
    queryFn: async () => {
      const { data, error } = await client.GET('/api/v1/prds', {
        params: { query: params },
      });
      if (error) throw error;
      return data;
    },
  });
}

// 查询单个PRD
export function usePRD(id: string) {
  return useQuery({
    queryKey: ['prds', id],
    queryFn: async () => {
      const { data, error } = await client.GET('/api/v1/prds/{id}', {
        params: { path: { id } },
      });
      if (error) throw error;
      return data;
    },
    enabled: !!id,
  });
}

// 创建PRD
export function useCreatePRD() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (dto: CreatePRDDto) => {
      const { data, error } = await client.POST('/api/v1/prds', {
        body: dto,
      });
      if (error) throw error;
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['prds'] });
    },
  });
}

// 更新PRD
export function useUpdatePRD(id: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (dto: UpdatePRDDto) => {
      const { data, error } = await client.PUT('/api/v1/prds/{id}', {
        params: { path: { id } },
        body: dto,
      });
      if (error) throw error;
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['prds', id] });
      queryClient.invalidateQueries({ queryKey: ['prds'] });
    },
  });
}

// 删除PRD
export function useDeletePRD() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: string) => {
      const { error } = await client.DELETE('/api/v1/prds/{id}', {
        params: { path: { id } },
      });
      if (error) throw error;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['prds'] });
    },
  });
}
```

---

## UI组件库集成

### 1. 安装shadcn/ui
```bash
npx shadcn@latest init
# ✔ Which style would you like to use? Default
# ✔ Which color would you like to use as base color? Slate
# ✔ Would you like to use CSS variables for colors? Yes
```

### 2. 添加常用组件
```bash
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add input
npx shadcn@latest add label
npx shadcn@latest add textarea
npx shadcn@latest add select
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
npx shadcn@latest add badge
npx shadcn@latest add separator
npx shadcn@latest add tabs
npx shadcn@latest add toast
```

### 3. 配置Tailwind（tailwind.config.ts）
```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: ['class'],
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        // ... 其他颜色
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
};

export default config;
```

---

## Dashboard布局设计

### 1. Dashboard布局（src/app/(dashboard)/layout.tsx）
```typescript
import { Header } from '@/components/layout/header';
import { Sidebar } from '@/components/layout/sidebar';

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex h-screen overflow-hidden">
      {/* 侧边栏 */}
      <Sidebar />

      {/* 主内容区 */}
      <div className="flex flex-1 flex-col overflow-hidden">
        {/* 顶部导航 */}
        <Header />

        {/* 页面内容 */}
        <main className="flex-1 overflow-y-auto bg-gray-50 p-6">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### 2. 侧边栏组件（src/components/layout/sidebar.tsx）
```typescript
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { cn } from '@/lib/utils';
import {
  FileText,
  Palette,
  Bug,
  Activity,
  Settings,
  LayoutDashboard,
} from 'lucide-react';

const navigation = [
  { name: '概览', href: '/', icon: LayoutDashboard },
  { name: 'PRD', href: '/prds', icon: FileText },
  { name: '设计稿', href: '/designs', icon: Palette },
  { name: 'Bug追踪', href: '/bugs', icon: Bug },
  { name: '活动流', href: '/activities', icon: Activity },
  { name: '设置', href: '/settings', icon: Settings },
];

export function Sidebar() {
  const pathname = usePathname();

  return (
    <aside className="hidden w-64 border-r bg-white lg:block">
      <div className="flex h-full flex-col">
        {/* Logo */}
        <div className="flex h-16 items-center border-b px-6">
          <h1 className="text-xl font-bold">TeamOS</h1>
        </div>

        {/* 导航菜单 */}
        <nav className="flex-1 space-y-1 p-4">
          {navigation.map((item) => {
            const isActive = pathname === item.href;
            return (
              <Link
                key={item.name}
                href={item.href}
                className={cn(
                  'flex items-center gap-3 rounded-lg px-3 py-2 text-sm font-medium transition-colors',
                  isActive
                    ? 'bg-primary text-primary-foreground'
                    : 'text-muted-foreground hover:bg-accent hover:text-accent-foreground',
                )}
              >
                <item.icon className="h-5 w-5" />
                {item.name}
              </Link>
            );
          })}
        </nav>

        {/* 用户信息 */}
        <div className="border-t p-4">
          <div className="flex items-center gap-3">
            <div className="h-10 w-10 rounded-full bg-primary/10" />
            <div className="flex-1">
              <p className="text-sm font-medium">用户名</p>
              <p className="text-xs text-muted-foreground">user@example.com</p>
            </div>
          </div>
        </div>
      </div>
    </aside>
  );
}
```

### 3. 顶部导航（src/components/layout/header.tsx）
```typescript
'use client';

import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Bell, Search, Menu } from 'lucide-react';

export function Header() {
  return (
    <header className="flex h-16 items-center justify-between border-b bg-white px-6">
      {/* 移动端菜单按钮 */}
      <Button variant="ghost" size="icon" className="lg:hidden">
        <Menu className="h-5 w-5" />
      </Button>

      {/* 搜索框 */}
      <div className="flex flex-1 items-center gap-4 lg:ml-0">
        <div className="relative w-full max-w-md">
          <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
          <Input
            type="search"
            placeholder="搜索PRD、设计稿、Bug..."
            className="pl-10"
          />
        </div>
      </div>

      {/* 右侧操作 */}
      <div className="flex items-center gap-2">
        <Button variant="ghost" size="icon">
          <Bell className="h-5 w-5" />
        </Button>
      </div>
    </header>
  );
}
```

---

## 核心页面实现

### 1. PRD列表页（src/app/(dashboard)/prds/page.tsx）
```typescript
'use client';

import { usePRDs } from '@/lib/hooks/use-prds';
import { PRDCard } from '@/components/prd/prd-card';
import { Button } from '@/components/ui/button';
import { Plus } from 'lucide-react';
import Link from 'next/link';

export default function PRDsPage() {
  const { data, isLoading, error } = usePRDs({ page: 1, pageSize: 20 });

  if (isLoading) {
    return <div>加载中...</div>;
  }

  if (error) {
    return <div>加载失败: {error.message}</div>;
  }

  return (
    <div className="space-y-6">
      {/* 页面标题 */}
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-3xl font-bold">PRD管理</h1>
          <p className="text-muted-foreground">管理产品需求文档</p>
        </div>
        <Link href="/prds/new">
          <Button>
            <Plus className="mr-2 h-4 w-4" />
            创建PRD
          </Button>
        </Link>
      </div>

      {/* PRD列表 */}
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {data?.items?.map((prd) => (
          <PRDCard key={prd.id} prd={prd} />
        ))}
      </div>

      {/* 分页 */}
      {data && data.totalPages > 1 && (
        <div className="flex justify-center gap-2">
          {/* 分页组件实现 */}
        </div>
      )}
    </div>
  );
}
```

### 2. PRD卡片组件（src/components/prd/prd-card.tsx）
```typescript
import Link from 'next/link';
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { formatDistanceToNow } from 'date-fns';
import { zhCN } from 'date-fns/locale';
import type { components } from '@/lib/api/types';

type PRD = components['schemas']['PRDResponseDto'];

const statusColors = {
  DRAFT: 'bg-gray-100 text-gray-800',
  REVIEW: 'bg-blue-100 text-blue-800',
  APPROVED: 'bg-green-100 text-green-800',
  REJECTED: 'bg-red-100 text-red-800',
  ARCHIVED: 'bg-gray-100 text-gray-600',
};

const priorityColors = {
  LOW: 'bg-gray-100 text-gray-800',
  MEDIUM: 'bg-yellow-100 text-yellow-800',
  HIGH: 'bg-orange-100 text-orange-800',
  URGENT: 'bg-red-100 text-red-800',
};

export function PRDCard({ prd }: { prd: PRD }) {
  return (
    <Card className="hover:shadow-lg transition-shadow">
      <CardHeader>
        <div className="flex items-start justify-between">
          <CardTitle className="line-clamp-2">{prd.title}</CardTitle>
          <Badge className={statusColors[prd.status]}>{prd.status}</Badge>
        </div>
        {prd.summary && (
          <CardDescription className="line-clamp-2">{prd.summary}</CardDescription>
        )}
      </CardHeader>

      <CardContent>
        <div className="flex items-center gap-2">
          <Badge variant="outline" className={priorityColors[prd.priority]}>
            {prd.priority}
          </Badge>
          <span className="text-xs text-muted-foreground">
            {formatDistanceToNow(new Date(prd.createdAt), {
              addSuffix: true,
              locale: zhCN,
            })}
          </span>
        </div>
      </CardContent>

      <CardFooter>
        <Link href={`/prds/${prd.id}`} className="w-full">
          <Button variant="outline" className="w-full">
            查看详情
          </Button>
        </Link>
      </CardFooter>
    </Card>
  );
}
```

### 3. PRD详情页（src/app/(dashboard)/prds/[id]/page.tsx）
```typescript
'use client';

import { usePRD } from '@/lib/hooks/use-prds';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Separator } from '@/components/ui/separator';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import { format } from 'date-fns';
import { zhCN } from 'date-fns/locale';
import { Edit, Trash2 } from 'lucide-react';

export default function PRDDetailPage({ params }: { params: { id: string } }) {
  const { data: prd, isLoading } = usePRD(params.id);

  if (isLoading) {
    return <div>加载中...</div>;
  }

  if (!prd) {
    return <div>PRD不存在</div>;
  }

  return (
    <div className="space-y-6">
      {/* 标题栏 */}
      <div className="flex items-start justify-between">
        <div className="space-y-2">
          <h1 className="text-3xl font-bold">{prd.title}</h1>
          <div className="flex items-center gap-2">
            <Badge>{prd.status}</Badge>
            <Badge variant="outline">{prd.priority}</Badge>
            <span className="text-sm text-muted-foreground">
              创建于 {format(new Date(prd.createdAt), 'PPP', { locale: zhCN })}
            </span>
          </div>
        </div>

        <div className="flex gap-2">
          <Button variant="outline">
            <Edit className="mr-2 h-4 w-4" />
            编辑
          </Button>
          <Button variant="destructive">
            <Trash2 className="mr-2 h-4 w-4" />
            删除
          </Button>
        </div>
      </div>

      <Separator />

      {/* 内容区 */}
      <Tabs defaultValue="content">
        <TabsList>
          <TabsTrigger value="content">内容</TabsTrigger>
          <TabsTrigger value="activities">活动历史</TabsTrigger>
        </TabsList>

        <TabsContent value="content" className="space-y-4">
          {prd.summary && (
            <div className="rounded-lg bg-muted p-4">
              <h3 className="font-semibold">摘要</h3>
              <p className="text-muted-foreground">{prd.summary}</p>
            </div>
          )}

          <div className="prose max-w-none">
            <ReactMarkdown remarkPlugins={[remarkGfm]}>
              {prd.content}
            </ReactMarkdown>
          </div>
        </TabsContent>

        <TabsContent value="activities">
          {/* 活动流组件 */}
          <div className="space-y-4">
            {prd.activities?.map((activity) => (
              <div key={activity.id} className="rounded-lg border p-4">
                <p className="text-sm">{activity.type}</p>
                <p className="text-xs text-muted-foreground">
                  {format(new Date(activity.published), 'PPpp', { locale: zhCN })}
                </p>
              </div>
            ))}
          </div>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

## 响应式设计

### 移动端适配策略
```typescript
// 使用Tailwind响应式前缀
<div className="
  grid 
  grid-cols-1        // 移动端: 1列
  md:grid-cols-2     // 平板: 2列
  lg:grid-cols-3     // 桌面: 3列
  gap-4
">
  {/* 卡片列表 */}
</div>

// 侧边栏在移动端隐藏
<aside className="hidden lg:block">
  {/* 侧边栏内容 */}
</aside>

// 移动端显示菜单按钮
<Button className="lg:hidden">
  <Menu />
</Button>
```

---

**前端基础架构已完成!继续阅读 [集成与部署](./03-integration-deploy.md)** 🎨


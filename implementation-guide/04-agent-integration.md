# AI Agent集成指南

> **多Agent协作、接口规范、错误处理完整方案**

---

## 📋 目录
- [Agent角色定义](#agent角色定义)
- [OpenAPI契约集成](#openapi契约集成)
- [AutoGen集成方案](#autogen集成方案)
- [CrewAI集成方案](#crewai集成方案)
- [协作流程设计](#协作流程设计)
- [错误处理与重试](#错误处理与重试)
- [幂等性保证](#幂等性保证)
- [可观测性](#可观测性)

---

## Agent角色定义

### 1. 角色职责矩阵

| Agent角色 | 主要职责 | 输入 | 输出 | 调用API |
|-----------|----------|------|------|---------|
| **PM Agent** | 需求分析、PRD编写 | 用户需求描述 | PRD文档 | POST /api/v1/prds |
| **Design Agent** | UI/UX设计、设计稿生成 | PRD文档 | 设计稿URL | POST /api/v1/designs |
| **Frontend Agent** | 前端代码实现 | 设计稿、API规范 | 代码提交记录 | GET /api/v1/designs/{id} |
| **Backend Agent** | API实现、数据库设计 | PRD、API规范 | 代码提交记录 | GET /api/v1/prds/{id} |
| **QA Agent** | 测试用例生成、Bug报告 | PRD、代码 | Bug列表 | POST /api/v1/bugs |
| **Build Agent** | CI/CD配置、构建部署 | 代码仓库 | 构建状态 | POST /api/v1/builds |
| **Security Agent** | 安全扫描、漏洞修复 | 代码、依赖 | 安全报告 | POST /api/v1/security-scans |

### 2. Agent用户创建（数据库初始化）
```sql
-- 创建Agent用户
INSERT INTO users (id, email, name, password, role) VALUES
  ('agent-pm', 'pm@agent.teamos', 'PM Agent', 'hashed-password', 'AGENT'),
  ('agent-design', 'design@agent.teamos', 'Design Agent', 'hashed-password', 'AGENT'),
  ('agent-frontend', 'frontend@agent.teamos', 'Frontend Agent', 'hashed-password', 'AGENT'),
  ('agent-backend', 'backend@agent.teamos', 'Backend Agent', 'hashed-password', 'AGENT'),
  ('agent-qa', 'qa@agent.teamos', 'QA Agent', 'hashed-password', 'AGENT'),
  ('agent-build', 'build@agent.teamos', 'Build Agent', 'hashed-password', 'AGENT'),
  ('agent-security', 'security@agent.teamos', 'Security Agent', 'hashed-password', 'AGENT');
```

---

## OpenAPI契约集成

### 1. 获取OpenAPI规范
```bash
# 后端服务启动后访问
curl http://localhost:3000/api/openapi.json > teamos-openapi.json
```

### 2. Agent工具封装（Python示例）
```python
import requests
from typing import Dict, Any, Optional

class TeamOSClient:
    """TeamOS API客户端，供Agent调用"""
    
    def __init__(self, base_url: str, token: str):
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        }
    
    def create_prd(self, title: str, content: str, 
                   summary: Optional[str] = None,
                   priority: str = "MEDIUM") -> Dict[str, Any]:
        """创建PRD"""
        response = requests.post(
            f"{self.base_url}/api/v1/prds",
            json={
                "title": title,
                "content": content,
                "summary": summary,
                "priority": priority
            },
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def get_prd(self, prd_id: str) -> Dict[str, Any]:
        """获取PRD详情"""
        response = requests.get(
            f"{self.base_url}/api/v1/prds/{prd_id}",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def update_prd(self, prd_id: str, **kwargs) -> Dict[str, Any]:
        """更新PRD"""
        response = requests.put(
            f"{self.base_url}/api/v1/prds/{prd_id}",
            json=kwargs,
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def create_design(self, title: str, file_url: str,
                      prd_id: Optional[str] = None) -> Dict[str, Any]:
        """创建设计稿"""
        response = requests.post(
            f"{self.base_url}/api/v1/designs",
            json={
                "title": title,
                "fileUrl": file_url,
                "prdId": prd_id
            },
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def create_bug(self, title: str, description: str,
                   severity: str = "MEDIUM") -> Dict[str, Any]:
        """创建Bug"""
        response = requests.post(
            f"{self.base_url}/api/v1/bugs",
            json={
                "title": title,
                "description": description,
                "severity": severity
            },
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def get_activities(self, page: int = 1, 
                       page_size: int = 20) -> Dict[str, Any]:
        """获取活动流"""
        response = requests.get(
            f"{self.base_url}/api/v1/activities",
            params={"page": page, "pageSize": page_size},
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
```

---

## AutoGen集成方案

### 1. 安装AutoGen
```bash
pip install autogen-agentchat autogen-ext
```

### 2. PM Agent实现
```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_ext.models import OpenAIChatCompletionClient

# 初始化API客户端
teamos_client = TeamOSClient(
    base_url="http://localhost:3000",
    token="agent-pm-token"
)

# 定义工具函数
async def create_prd_tool(title: str, content: str, 
                          summary: str = None, 
                          priority: str = "MEDIUM") -> str:
    """创建PRD文档
    
    Args:
        title: PRD标题
        content: PRD详细内容（Markdown格式）
        summary: PRD摘要（可选）
        priority: 优先级（LOW/MEDIUM/HIGH/URGENT）
    
    Returns:
        创建成功的PRD ID
    """
    result = teamos_client.create_prd(
        title=title,
        content=content,
        summary=summary,
        priority=priority
    )
    return f"PRD创建成功，ID: {result['id']}"

# 创建PM Agent
pm_agent = AssistantAgent(
    name="PM_Agent",
    model_client=OpenAIChatCompletionClient(
        model="gpt-4",
        api_key="your-openai-key"
    ),
    tools=[create_prd_tool],
    system_message="""你是一个专业的产品经理Agent。
    
    职责：
    1. 分析用户需求，提炼核心功能点
    2. 编写结构化的PRD文档（包含背景、目标、功能、验收标准）
    3. 使用Markdown格式组织内容
    4. 调用create_prd_tool创建PRD
    
    输出格式：
    ## 背景
    ## 目标
    ## 功能需求
    ## 验收标准
    """
)
```

### 3. 多Agent协作流程
```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.ui import Console

# 创建Design Agent
design_agent = AssistantAgent(
    name="Design_Agent",
    model_client=OpenAIChatCompletionClient(model="gpt-4"),
    tools=[create_design_tool],
    system_message="你是UI/UX设计师，根据PRD生成设计方案..."
)

# 创建QA Agent
qa_agent = AssistantAgent(
    name="QA_Agent",
    model_client=OpenAIChatCompletionClient(model="gpt-4"),
    tools=[create_bug_tool],
    system_message="你是QA工程师，负责测试和Bug报告..."
)

# 创建协作团队
team = RoundRobinGroupChat(
    participants=[pm_agent, design_agent, qa_agent],
    termination_condition=lambda messages: "TERMINATE" in messages[-1].content
)

# 运行协作流程
async def run_workflow(user_requirement: str):
    """运行完整的Agent协作流程"""
    result = await Console(team.run_stream(
        task=f"用户需求：{user_requirement}\n\n请PM Agent先创建PRD，然后Design Agent生成设计方案。"
    ))
    return result

# 示例调用
import asyncio
asyncio.run(run_workflow("开发一个用户登录功能"))
```

---

## CrewAI集成方案

### 1. 安装CrewAI
```bash
pip install crewai crewai-tools
```

### 2. 定义Agent和Task
```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import tool

# 定义工具
@tool("Create PRD")
def create_prd_crewai(title: str, content: str) -> str:
    """创建PRD文档"""
    result = teamos_client.create_prd(title=title, content=content)
    return f"PRD ID: {result['id']}"

@tool("Create Design")
def create_design_crewai(title: str, file_url: str, prd_id: str) -> str:
    """创建设计稿"""
    result = teamos_client.create_design(
        title=title, 
        file_url=file_url, 
        prd_id=prd_id
    )
    return f"Design ID: {result['id']}"

# 创建PM Agent
pm_agent = Agent(
    role="Product Manager",
    goal="分析需求并创建结构化的PRD文档",
    backstory="你是一位经验丰富的产品经理，擅长需求分析和文档编写。",
    tools=[create_prd_crewai],
    verbose=True
)

# 创建Design Agent
design_agent = Agent(
    role="UI/UX Designer",
    goal="根据PRD生成设计方案",
    backstory="你是一位专业的UI/UX设计师，擅长用户体验设计。",
    tools=[create_design_crewai],
    verbose=True
)

# 定义任务
prd_task = Task(
    description="分析用户需求：{requirement}，创建PRD文档",
    expected_output="PRD ID和摘要",
    agent=pm_agent
)

design_task = Task(
    description="根据PRD创建UI设计方案",
    expected_output="设计稿ID和预览链接",
    agent=design_agent,
    context=[prd_task]  # 依赖PRD任务
)

# 创建Crew
crew = Crew(
    agents=[pm_agent, design_agent],
    tasks=[prd_task, design_task],
    process=Process.sequential,  # 顺序执行
    verbose=True
)

# 运行
result = crew.kickoff(inputs={"requirement": "用户登录功能"})
print(result)
```

---

## 协作流程设计

### 1. 标准工作流
```
用户输入需求
    ↓
PM Agent 分析需求 → 创建PRD
    ↓
Design Agent 读取PRD → 生成设计稿
    ↓
Frontend Agent 读取设计稿 → 实现前端
    ↓
Backend Agent 读取PRD → 实现API
    ↓
QA Agent 测试功能 → 报告Bug
    ↓
Build Agent 构建部署 → 记录状态
    ↓
Security Agent 安全扫描 → 生成报告
```

### 2. 流程编排（Python）
```python
async def full_workflow(requirement: str):
    """完整的开发流程"""
    
    # 1. PM创建PRD
    print("📝 PM Agent 正在分析需求...")
    prd = teamos_client.create_prd(
        title=f"需求：{requirement}",
        content=await pm_agent.generate_prd(requirement)
    )
    print(f"✅ PRD创建完成: {prd['id']}")
    
    # 2. Design生成设计稿
    print("🎨 Design Agent 正在设计UI...")
    design = teamos_client.create_design(
        title=f"{prd['title']} - 设计稿",
        file_url=await design_agent.generate_design(prd['id']),
        prd_id=prd['id']
    )
    print(f"✅ 设计稿完成: {design['id']}")
    
    # 3. 并行开发（前端+后端）
    print("💻 Frontend & Backend Agent 正在开发...")
    await asyncio.gather(
        frontend_agent.implement(design['id']),
        backend_agent.implement(prd['id'])
    )
    print("✅ 开发完成")
    
    # 4. QA测试
    print("🧪 QA Agent 正在测试...")
    bugs = await qa_agent.test(prd['id'])
    if bugs:
        print(f"⚠️ 发现 {len(bugs)} 个Bug")
        for bug in bugs:
            teamos_client.create_bug(**bug)
    else:
        print("✅ 测试通过")
    
    # 5. 构建部署
    print("🚀 Build Agent 正在部署...")
    await build_agent.deploy()
    print("✅ 部署完成")
    
    # 6. 安全扫描
    print("🔒 Security Agent 正在扫描...")
    await security_agent.scan()
    print("✅ 安全检查完成")
    
    return {
        "prd_id": prd['id'],
        "design_id": design['id'],
        "status": "completed"
    }
```

---

## 错误处理与重试

### 1. 指数退避重试
```python
import time
from typing import Callable, Any

def retry_with_backoff(
    func: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0
) -> Any:
    """指数退避重试装饰器"""
    for attempt in range(max_retries):
        try:
            return func()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            
            delay = min(base_delay * (2 ** attempt), max_delay)
            print(f"⚠️ 请求失败，{delay}秒后重试... (尝试 {attempt + 1}/{max_retries})")
            time.sleep(delay)

# 使用示例
def create_prd_with_retry(title: str, content: str):
    return retry_with_backoff(
        lambda: teamos_client.create_prd(title, content)
    )
```

### 2. 错误分类处理
```python
class AgentError(Exception):
    """Agent错误基类"""
    pass

class RetryableError(AgentError):
    """可重试错误（网络、超时等）"""
    pass

class FatalError(AgentError):
    """致命错误（认证失败、数据验证失败等）"""
    pass

def handle_api_error(error: requests.exceptions.RequestException):
    """错误分类处理"""
    if isinstance(error, requests.exceptions.Timeout):
        raise RetryableError("请求超时") from error
    elif isinstance(error, requests.exceptions.ConnectionError):
        raise RetryableError("连接失败") from error
    elif hasattr(error, 'response') and error.response.status_code == 401:
        raise FatalError("认证失败") from error
    elif hasattr(error, 'response') and error.response.status_code == 422:
        raise FatalError("数据验证失败") from error
    else:
        raise RetryableError("未知错误") from error
```

---

## 幂等性保证

### 1. 使用Idempotency-Key
```python
import uuid

class IdempotentClient(TeamOSClient):
    """支持幂等性的客户端"""
    
    def create_prd_idempotent(self, title: str, content: str, 
                              idempotency_key: str = None) -> Dict[str, Any]:
        """幂等创建PRD"""
        if not idempotency_key:
            idempotency_key = str(uuid.uuid4())
        
        headers = {
            **self.headers,
            "Idempotency-Key": idempotency_key
        }
        
        response = requests.post(
            f"{self.base_url}/api/v1/prds",
            json={"title": title, "content": content},
            headers=headers
        )
        response.raise_for_status()
        return response.json()
```

### 2. 后端幂等性实现（NestJS）
```typescript
// src/common/interceptors/idempotency.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  private cache = new Map<string, any>();

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const idempotencyKey = request.headers['idempotency-key'];

    if (!idempotencyKey) {
      return next.handle();
    }

    // 检查缓存
    if (this.cache.has(idempotencyKey)) {
      return new Observable((observer) => {
        observer.next(this.cache.get(idempotencyKey));
        observer.complete();
      });
    }

    // 执行并缓存结果
    return next.handle().pipe(
      tap((data) => {
        this.cache.set(idempotencyKey, data);
        // 1小时后清除缓存
        setTimeout(() => this.cache.delete(idempotencyKey), 3600000);
      }),
    );
  }
}
```

---

## 可观测性

### 1. Agent活动追踪
```python
import logging
from datetime import datetime

class ObservableAgent:
    """可观测的Agent基类"""
    
    def __init__(self, name: str, client: TeamOSClient):
        self.name = name
        self.client = client
        self.logger = logging.getLogger(name)
    
    def log_activity(self, action: str, details: dict):
        """记录Agent活动"""
        self.logger.info(f"{action}: {details}")
        
        # 同时记录到ActivityStreams
        # （后端自动记录，这里仅作示例）
    
    async def execute_task(self, task: str):
        """执行任务并记录"""
        start_time = datetime.now()
        self.log_activity("task_started", {"task": task})
        
        try:
            result = await self._do_task(task)
            duration = (datetime.now() - start_time).total_seconds()
            self.log_activity("task_completed", {
                "task": task,
                "duration": duration,
                "result": result
            })
            return result
        except Exception as e:
            self.log_activity("task_failed", {
                "task": task,
                "error": str(e)
            })
            raise
```

### 2. 监控Dashboard查询
```python
def get_agent_metrics(agent_id: str, days: int = 7):
    """获取Agent性能指标"""
    activities = teamos_client.get_activities(
        actor_id=agent_id,
        start_date=(datetime.now() - timedelta(days=days)).isoformat()
    )
    
    metrics = {
        "total_actions": len(activities['items']),
        "action_types": {},
        "success_rate": 0,
        "avg_duration": 0
    }
    
    for activity in activities['items']:
        action_type = activity['type']
        metrics['action_types'][action_type] = \
            metrics['action_types'].get(action_type, 0) + 1
    
    return metrics
```

---

**AI Agent集成方案已完成！继续阅读 [ActivityStreams实现](./05-activitystreams.md)** 🤖


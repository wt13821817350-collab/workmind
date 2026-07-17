# WorkMind API 设计文档

> 版本：v0.1 | 状态：草稿 | 日期：2026-07-16
> 关联文档：[系统设计文档](../architecture/system-design.zh-CN.md) · [数据库设计](../database/schema.md)

---

## 1. REST API 规范

### 1.1 URL 命名规范

- 所有路径小写字母加连字符，不使用下划线或驼峰：`/api/v1/knowledge-bases`
- 资源名使用复数名词：`/users`、`/agents`、`/documents`
- 层级关系通过路径嵌套表达（最多两层）：`/knowledge-bases/{kb_id}/documents`
- 不在 URL 中使用动词；操作类接口用 POST + 动词路径：`POST /api/v1/agents/{id}/execute`

### 1.2 HTTP 方法

| 方法 | 用途 | 示例 |
|---|---|---|
| GET | 查询资源 | GET /api/v1/agents |
| POST | 创建资源 / 触发操作 | POST /api/v1/agents |
| PUT | 全量更新资源 | PUT /api/v1/agents/{id} |
| PATCH | 部分更新资源 | PATCH /api/v1/agents/{id}/status |
| DELETE | 删除资源 | DELETE /api/v1/agents/{id} |

### 1.3 版本控制

所有接口以 `/api/v1/` 为前缀。版本升级时新增 `/api/v2/`，旧版本至少维护 6 个月。

### 1.4 统一响应格式

```json
{ "code": 0, "message": "ok", "data": { } }
```

**分页响应（列表接口）：**
```json
{
  "code": 0, "message": "ok",
  "data": { "items": [], "total": 100, "page": 1, "page_size": 20 }
}
```

**错误响应：**
```json
{ "code": 40300, "message": "权限不足", "data": null }
```

### 1.5 错误码规范

| 错误码 | 含义 |
|---|---|
| 0 | 成功 |
| 40001 | 用户名或密码错误 |
| 40100 | 未登录 / Token 无效 |
| 40101 | Token 已过期，请刷新 |
| 40300 | 权限不足 |
| 40400 | 资源不存在 |
| 40900 | 资源冲突（如用户名重复） |
| 42200 | 请求参数校验失败 |
| 50000 | 服务内部错误 |
| 50001 | AI 服务调用失败 |
| 50002 | 模型响应超时 |

### 1.6 分页规范

| 参数 | 说明 | 默认值 |
|---|---|---|
| page | 页码（从 1 开始） | 1 |
| page_size | 每页条数（最大 100） | 20 |
| order_by | 排序字段 | created_at |
| order | 排序方向（asc / desc） | desc |

### 1.7 认证规范

所有接口（除 `/auth/login` 外）需在请求头携带 JWT：

```
Authorization: Bearer <access_token>
```

Token 过期（40101）时，调用 `POST /api/v1/auth/refresh` 换取新 Token，无需用户重新登录。

---

## 2. 认证接口（Auth）

### POST /api/v1/auth/login

**Request:**
```json
{ "username": "admin", "password": "YourPassword" }
```

**Response:**
```json
{
  "code": 0,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
    "expires_in": 7200,
    "user": { "id": 1, "username": "admin", "email": "admin@workmind.ai", "org_id": 1 }
  }
}
```

### POST /api/v1/auth/logout

使当前 Token 失效（服务端写入 Redis 黑名单，至 Token 自然过期为止）。

**Headers:** `Authorization: Bearer <token>`
**Response:** `{ "code": 0, "message": "ok", "data": null }`

### POST /api/v1/auth/refresh

**Request:** `{ "refresh_token": "eyJ..." }`
**Response:** 同 login，返回新的 access_token 和 refresh_token。

### GET /api/v1/auth/me

返回当前用户信息及权限码列表，前端用于菜单渲染和按钮级权限控制。

**Response:**
```json
{
  "code": 0,
  "data": {
    "id": 1, "username": "admin", "org_id": 1,
    "roles": ["admin"],
    "permissions": ["knowledge:document:delete", "agent:execute", "workflow:run"]
  }
}
```

---

## 3. RBAC 接口

### 用户管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/users | 用户列表（支持按部门/角色筛选） |
| POST | /api/v1/users | 创建用户 |
| GET | /api/v1/users/{id} | 用户详情 |
| PUT | /api/v1/users/{id} | 更新用户信息 |
| PATCH | /api/v1/users/{id}/status | 启用 / 禁用用户 |
| DELETE | /api/v1/users/{id} | 删除用户（软删除） |
| PUT | /api/v1/users/{id}/roles | 覆盖式分配角色 |

**创建用户 Request:**
```json
{ "username": "lisi", "email": "lisi@company.com", "password": "InitPass123", "dept_id": 3, "role_ids": [2] }
```

### 角色管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/roles | 角色列表 |
| POST | /api/v1/roles | 创建角色 |
| GET | /api/v1/roles/{id} | 角色详情（含权限列表） |
| PUT | /api/v1/roles/{id} | 更新角色 |
| DELETE | /api/v1/roles/{id} | 删除角色 |
| PUT | /api/v1/roles/{id}/permissions | 覆盖式配置角色权限 |

### 权限列表

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/permissions | 全量权限列表（按模块分组返回） |

**Response:**
```json
{
  "code": 0,
  "data": [
    { "module": "knowledge", "items": [
        { "id": 1, "code": "knowledge:base:create", "type": "api" },
        { "id": 2, "code": "knowledge:document:delete", "type": "api" }
    ]},
    { "module": "agent", "items": [
        { "id": 8, "code": "agent:execute", "type": "api" }
    ]}
  ]
}
```

### 部门管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/departments | 部门树（递归结构） |
| POST | /api/v1/departments | 创建部门 |
| PUT | /api/v1/departments/{id} | 更新部门 |
| DELETE | /api/v1/departments/{id} | 删除部门（有成员时拒绝） |

---

## 4. 知识库接口

### 知识库管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/knowledge-bases | 知识库列表 |
| POST | /api/v1/knowledge-bases | 创建知识库 |
| GET | /api/v1/knowledge-bases/{kb_id} | 知识库详情（含文档统计） |
| PUT | /api/v1/knowledge-bases/{kb_id} | 更新知识库 |
| DELETE | /api/v1/knowledge-bases/{kb_id} | 删除知识库（含下属所有文档） |

### 文档管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/knowledge-bases/{kb_id}/documents | 文档列表（含处理状态） |
| POST | /api/v1/knowledge-bases/{kb_id}/documents | 上传文档（multipart/form-data） |
| GET | /api/v1/knowledge-bases/{kb_id}/documents/{doc_id} | 文档详情 |
| DELETE | /api/v1/knowledge-bases/{kb_id}/documents/{doc_id} | 删除文档 |
| POST | /api/v1/knowledge-bases/{kb_id}/documents/{doc_id}/reprocess | 重新处理（失败后重试） |

上传文档后异步处理（切片 → Embedding → 写入向量库），客户端轮询 `status`：`pending → processing → completed / failed`。

---

## 5. RAG 接口

### POST /api/v1/knowledge-bases/{kb_id}/search

语义检索，返回最相关切片列表，**不调用 LLM**，供调试或自定义展示使用。

**Request:**
```json
{ "query": "退货政策是什么", "top_k": 5, "score_threshold": 0.7 }
```

**Response:**
```json
{
  "code": 0,
  "data": { "chunks": [
    { "chunk_id": 1024, "document_name": "售后政策.pdf", "content": "凡在本平台购买...", "score": 0.92 }
  ]}
}
```

### POST /api/v1/knowledge-bases/{kb_id}/ask

RAG 问答，调用 LLM 生成引用原文的回答，默认流式输出（SSE）。

**Request:**
```json
{ "question": "退货政策是什么", "conversation_id": 101, "model_id": 2, "stream": true }
```

**Response:** `Content-Type: text/event-stream`

```
data: {"type":"chunk","content":"根据公司退货政策，"}
data: {"type":"chunk","content":"自收货之日起7天内可申请无理由退换货..."}
data: {"type":"sources","chunks":[{"chunk_id":1024,"document_name":"售后政策.pdf","score":0.92}]}
data: {"type":"done","message_id":5021,"total_tokens":312}
```

`sources` 事件记录本次引用的切片 ID，写入 `message.retrieved_chunks`，支撑 AI 回答可追溯要求。

---

## 6. Agent 接口

### Agent 管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/agents | Agent 列表 |
| POST | /api/v1/agents | 创建 Agent |
| GET | /api/v1/agents/{agent_id} | Agent 详情（含绑定工具列表） |
| PUT | /api/v1/agents/{agent_id} | 更新 Agent |
| PATCH | /api/v1/agents/{agent_id}/status | 启用 / 禁用 |
| DELETE | /api/v1/agents/{agent_id} | 删除 Agent |
| PUT | /api/v1/agents/{agent_id}/tools | 覆盖式更新绑定工具 |

**创建 Agent Request:**
```json
{
  "name": "客服助手",
  "description": "处理用户售后问题",
  "model_id": 1,
  "system_prompt": "你是 WorkMind 客服助手，请基于知识库内容回答用户问题...",
  "tool_ids": [3, 7]
}
```

### Agent 执行

**POST /api/v1/agents/{agent_id}/execute**

`stream: false` 返回 execution_id 后异步执行；`stream: true` 时 SSE 实时返回执行过程。

**Request:**
```json
{ "input": "帮我生成本周销售分析报告并发邮件给老板", "conversation_id": 101, "stream": false }
```

**Response（非流式）:**
```json
{ "code": 0, "data": { "execution_id": 8801, "status": "running" } }
```

### 执行记录查询

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/agents/{agent_id}/executions | Agent 历史执行列表 |
| GET | /api/v1/executions/{execution_id} | 执行详情（含 token 消耗和成本） |
| GET | /api/v1/executions/{execution_id}/steps | 步骤列表（监控面板与 Debug 用） |

**执行详情 Response:**
```json
{
  "code": 0,
  "data": {
    "id": 8801, "status": "success", "total_tokens": 2150, "cost": 0.003,
    "started_at": "2026-07-16T10:00:00Z", "finished_at": "2026-07-16T10:00:12Z",
    "steps": [
      { "step_index": 1, "step_type": "planning",  "duration_ms": 320,  "status": "success" },
      { "step_index": 2, "step_type": "tool_call",  "tool_name": "query_database", "duration_ms": 1200, "status": "success" },
      { "step_index": 3, "step_type": "tool_call",  "tool_name": "generate_excel", "duration_ms": 860,  "status": "success" },
      { "step_index": 4, "step_type": "llm",         "tokens": 1540, "duration_ms": 4200, "status": "success" },
      { "step_index": 5, "step_type": "tool_call",  "tool_name": "send_email",     "duration_ms": 430,  "status": "success" }
    ]
  }
}
```

---

## 7. Workflow 接口

### Workflow 管理

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | /api/v1/workflows | Workflow 列表 |
| POST | /api/v1/workflows | 创建 Workflow |
| GET | /api/v1/workflows/{wf_id} | Workflow 详情（含 definition_json） |
| PUT | /api/v1/workflows/{wf_id} | 更新 Workflow |
| PATCH | /api/v1/workflows/{wf_id}/status | 启用 / 禁用 |
| DELETE | /api/v1/workflows/{wf_id} | 删除 Workflow |

**创建 Workflow Request:**
```json
{
  "name": "每日销售日报",
  "definition_json": {
    "trigger": { "type": "cron", "cron": "0 9 * * 1-5" },
    "nodes": [
      { "id": "n1", "type": "query_data",   "config": { "sql": "SELECT ..." } },
      { "id": "n2", "type": "llm_generate", "config": { "prompt": "基于以下数据生成日报：{{n1.output}}" } },
      { "id": "n3", "type": "send_message", "config": { "channel": "wecom", "group_id": "xxx" } }
    ],
    "edges": [{ "from": "n1", "to": "n2" }, { "from": "n2", "to": "n3" }]
  }
}
```

### Workflow 执行

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/workflows/{wf_id}/run | 手动触发执行 |
| GET | /api/v1/workflows/{wf_id}/tasks | 执行记录列表 |
| GET | /api/v1/workflow-tasks/{task_id} | 执行记录详情 |

**执行记录 Response:**
```json
{
  "code": 0,
  "data": {
    "id": 501, "workflow_id": 12, "trigger_type": "manual", "status": "success",
    "input_data": { "date_range": "2026-07-10~2026-07-16" },
    "output_data": { "report_url": "https://storage.workmind.ai/reports/..." },
    "started_at": "2026-07-16T09:00:00Z", "finished_at": "2026-07-16T09:00:08Z"
  }
}
```

---

## 附录：接口汇总

| 模块 | 接口数量 | 主要覆盖 |
|---|---|---|
| 认证 | 4 | Login / Logout / Refresh / Me |
| 用户与 RBAC | 16 | 用户、角色、权限、部门 |
| 知识库 | 9 | 知识库 CRUD + 文档上传 + 重处理 |
| RAG | 2 | 语义检索 + 问答（流式 SSE） |
| Agent | 10 | Agent CRUD + 执行 + 步骤查询 |
| Workflow | 8 | Workflow CRUD + 执行 + 任务记录 |
| **合计** | **49** | MVP 阶段接口总量 |

FastAPI 自动生成 OpenAPI 文档，部署后访问 `/docs` 查看交互式接口文档，`/redoc` 查看文档化版本。

---

*文档版本 v0.1，接口细节随开发迭代更新。完整请求/响应 Schema 以代码实现为准。*

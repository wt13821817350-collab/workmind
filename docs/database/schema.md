# WorkMind 数据库设计文档

> 版本：v0.2 | 状态：草稿 | 日期：2026-07-16
> 关联文档：[产品需求文档](../product/requirements.md) · [系统设计文档](../architecture/system-design.zh-CN.md)

---

## 1. 数据库概述（Database Overview）

WorkMind 采用 MySQL 作为主存储，承载所有结构化业务数据（用户、组织、权限、Workflow、日志等）；向量检索相关的 Embedding 数据存放在独立的向量数据库中（见[系统设计文档第七章](../architecture/system-design.zh-CN.md#第七章部署架构deployment-architecture)），两者通过 `document_chunk` 表中的引用字段关联。

本文档描述的是 MySQL 中的表结构设计，不包含向量数据库内部的索引结构（HNSW / IVF 等由向量数据库自身管理）。

数据库按业务领域分为六组：

| 分组 | 包含表 | 说明 |
|---|---|---|
| 用户与权限 | organization, department, user, role, permission, role_permission, user_role | 组织架构与 RBAC |
| 知识库 | knowledge_base, document, document_chunk | 知识管理与 RAG 检索 |
| 对话系统 | conversation, message, message_feedback | AI 对话与反馈 |
| AI 能力 | agent, agent_execution, execution_step, tool, agent_tool | Agent 执行与工具调用 |
| 模型与提示 | model_config, prompt_template | 模型配置与 Prompt 资产管理 |
| 工作流 | workflow, workflow_task | 流程编排与执行记录 |
| 系统 | system_log | 审计与操作日志 |

---

## 2. 数据库设计原则（Database Design Principles）

### 2.1 企业级多租户（Enterprise Multi Tenant）

支持多个组织（Organization）同时使用同一套系统。几乎所有业务表都带有 `org_id` 字段，作为数据归属的第一道边界。

### 2.2 数据隔离（Data Isolation）

不同组织之间的数据不能互相访问。查询层必须在每一次涉及业务数据的查询中携带 `org_id` 过滤条件，这一约束在应用层（ORM 中间件）统一强制，而不是依赖每个开发者手写时记得加。

### 2.3 可扩展（Extensible）

表结构要为未来的 Agent、Plugin、Multi-Agent 协作留出空间。凡是"配置类"的复杂结构（Agent 的系统提示词、Tool 的调用 Schema、Workflow 的节点定义）都用 `JSON` 字段承载，而不是拆成大量强类型列，避免每次加一个新能力都要改表结构。

### 2.4 可审计（Auditability）

重要操作必须被记录，且记录不可篡改、不可删除。`system_log` 表只允许插入（Append-only），不提供业务层的更新或删除接口。

---

## 3. ER 模型（ER Model）

```mermaid
erDiagram
    ORGANIZATION ||--o{ DEPARTMENT : has
    ORGANIZATION ||--o{ USER : has
    ORGANIZATION ||--o{ KNOWLEDGE_BASE : owns
    ORGANIZATION ||--o{ WORKFLOW : owns

    DEPARTMENT ||--o{ USER : contains
    USER }o--o{ ROLE : "user_role"
    ROLE }o--o{ PERMISSION : "role_permission"

    USER ||--o{ CONVERSATION : starts
    CONVERSATION ||--o{ MESSAGE : contains
    MESSAGE ||--o{ MESSAGE_FEEDBACK : receives

    KNOWLEDGE_BASE ||--o{ DOCUMENT : contains
    DOCUMENT ||--o{ DOCUMENT_CHUNK : "split into"

    USER ||--o{ AGENT : creates
    AGENT }o--o{ TOOL : "agent_tool"

    USER ||--o{ WORKFLOW : creates
    WORKFLOW ||--o{ WORKFLOW_TASK : "executed as"

    USER ||--o{ SYSTEM_LOG : triggers
```

**关键关系说明：**

- 一个组织下有多个部门、多个用户，知识库和 Workflow 也归属于组织，这是数据隔离的边界起点。
- 用户与角色是多对多（一个用户可以有多个角色），角色与权限也是多对多，中间通过 `user_role`、`role_permission` 两张关联表落地，这是标准 RBAC 设计。
- 一个知识库下有多篇文档，一篇文档切分成多个 Chunk，Chunk 是 RAG 检索真正命中的最小单元。
- 一个 Agent 可以绑定多个 Tool，一个 Tool 也可以被多个 Agent 复用，中间用 `agent_tool` 关联。
- Workflow 是定义（模板），Workflow Task 是每一次具体的执行记录，一对多。

---

## 4. 核心表设计（Core Tables）

### 4.1 用户与权限

#### organization（组织）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| name | 组织名称 |
| status | 状态（正常/禁用） |
| created_at | 创建时间 |

#### department（部门）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 部门名称 |
| parent_id | 上级部门（支持多级部门树） |

#### user（用户）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| dept_id | 所属部门 |
| username | 用户名 |
| password_hash | 密码哈希（不存明文） |
| email | 邮箱 |
| status | 状态（正常/禁用） |
| created_at | 创建时间 |

#### role（角色）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 角色名称 |
| description | 描述 |

#### permission（权限）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| code | 权限编码（如 `knowledge:document:delete`） |
| type | 类型（菜单 / API） |
| resource | 关联的菜单路径或 API 路径 |

#### user_role / role_permission（关联表）

标准多对多中间表，只存两侧外键 + 创建时间，不承载额外业务字段。

---

### 4.2 知识库

#### knowledge_base（知识库）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 知识库名称 |
| description | 描述 |
| owner_id | 创建者 |
| vector_store | 关联的向量数据库 collection 名称 |
| status | 状态 |

#### document（文档）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| kb_id | 所属知识库 |
| file_name | 文件名 |
| file_path | 存储路径（MinIO） |
| file_type | 文件类型（PDF / Word / TXT / 图片） |
| status | 处理状态（待处理 / 切片中 / 已完成 / 失败） |
| uploaded_by | 上传人 |
| created_at | 上传时间 |

#### document_chunk（文档切片）

这是 AI 项目里最容易被追问细节的一张表，因为它是"原文 → Chunk → Embedding → 向量数据库"这条链路在关系数据库侧的落点。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| document_id | 所属文档 |
| chunk_index | 切片顺序号（保证能还原原文顺序） |
| content | 切片文本内容 |
| token_count | 该切片的 Token 数（用于成本与检索质量分析） |
| vector_id | 该切片在向量数据库中的向量 ID（跨库引用，不存向量本身） |
| vector_store_type | 向量数据库类型（milvus / pgvector / chroma），支持未来多向量库路由 |
| created_at | 创建时间 |

**数据流转说明：**

```
原始文档（document）
    ↓ 切片
文档切片（document_chunk，存文本 + 顺序）
    ↓ Embedding
向量（存入向量数据库，document_chunk.vector_id 指回来源）
    ↓ 检索时
用户问题 → 向量检索 → 命中 vector_id → 反查 document_chunk.content → 拼入 Prompt
```

MySQL 里的 `document_chunk` 永远保留人类可读的原文，向量数据库只保留向量和最基本的元数据，两者通过 `vector_id` 单向引用，避免向量数据库成为唯一真源（一旦向量数据库需要重建索引，原文不会丢）。

---

### 4.3 对话系统

不按"简单聊天记录"来设计，因为后续 Agent 的 Memory、多轮任务追踪都要复用这套表。

#### conversation（会话）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| user_id | 所属用户 |
| agent_id | 关联的 Agent（可为空，纯聊天场景不绑定 Agent） |
| title | 会话标题 |
| status | 状态（进行中 / 已结束） |
| created_at | 创建时间 |

#### message（消息）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| conversation_id | 所属会话 |
| role | 角色（user / assistant / tool / system） |
| content | 消息内容 |
| tool_calls | 本条消息触发的工具调用记录（JSON，可为空） |
| retrieved_chunks | 本条回答引用的 document_chunk ID 列表（JSON，支撑"可追溯"原则） |
| created_at | 创建时间 |

#### message_feedback（消息反馈）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| message_id | 所属消息 |
| user_id | 反馈人 |
| rating | 评价（赞 / 踩） |
| comment | 补充说明 |
| created_at | 创建时间 |

`message.retrieved_chunks` 是 RAG 可追溯性在数据库层的直接体现——对应[产品需求文档第十一章](../product/requirements.md#第十一章产品原则product-principles)"所有 AI 回答必须可追溯"的原则：不是靠模型自己声明引用了什么，而是系统在生成回答的那一刻就把用到的 Chunk ID 记下来。

---

### 4.4 AI 相关表

这一组表是 WorkMind 和普通后台管理系统最大的区别所在。

#### agent（Agent）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | Agent 名称 |
| description | 描述 |
| model_id | 关联 model_config（不直接存模型名，通过模型配置表统一管理） |
| system_prompt | 系统提示词（当前生效版本，历史版本见未来扩展 agent_version） |
| status | 状态（启用 / 禁用） |
| creator_id | 创建人 |

#### tool（工具）

面向未来的 MCP 接入设计，一张表同时覆盖"内部工具"和"外部 MCP 工具"两种类型。`input_schema` / `output_schema` 拆开存，是因为 LLM 调用工具时必须读到 `description` 和 `input_schema` 才能决定是否调用——合并成一个 `schema` 字段意味着应用层每次都要解析才能取到这两个关键字段。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 工具名称 |
| description | 工具描述（LLM 用此判断何时调用该工具） |
| type | 类型（内置函数 / HTTP API / MCP Server） |
| endpoint | 调用地址（HTTP URL 或 MCP Server 地址，内置函数可为空） |
| input_schema | 输入参数的 JSON Schema |
| output_schema | 输出结构的 JSON Schema |
| auth_config | 鉴权配置（JSON，如 API Key、OAuth Token，敏感字段加密存储） |
| version | 工具版本号（MCP 工具版本管理） |
| status | 状态（启用 / 禁用） |

#### agent_tool（关联表）

支持一个 Agent 绑定多个 Tool，一个 Tool 被多个 Agent 复用。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| agent_id | Agent |
| tool_id | Tool |
| created_at | 绑定时间 |

---

### 4.7 模型配置与 Prompt 资产

#### model_config（模型配置）

企业使用多个模型是常态（DeepSeek 用于日常问答控制成本，Claude 用于复杂任务，内部私有化部署模型用于敏感数据）。把模型管理提升为独立表，Agent 通过 `model_id` 引用，而不是直接硬编码模型名字符串，切换模型时只需改配置，不需要更新每个 Agent 记录。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 配置名称（如"客服专用 DeepSeek"） |
| provider | 提供商（openai / anthropic / deepseek / qwen / custom） |
| model_name | 实际模型名称（如 deepseek-chat、claude-sonnet） |
| api_endpoint | API 地址（私有化部署或代理时可自定义） |
| api_key | API Key（加密存储） |
| status | 状态（启用 / 禁用） |
| created_at | 创建时间 |

#### prompt_template（Prompt 模板）

Prompt 是企业的核心资产，和代码一样需要版本管理和复用机制。Agent 可以引用模板，也可以自定义，模板库则沉淀企业最佳实践。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 模板名称 |
| content | Prompt 内容（支持变量占位符，如 `{{user_name}}`） |
| version | 版本号 |
| creator_id | 创建人 |
| status | 状态 |
| created_at | 创建时间 |

---

### 4.8 AI 执行追踪

这是 v0.1 版本缺失的最关键部分。WorkMind 定位是 AI 员工平台，一次 Agent 任务可能涉及多步工具调用和多次 LLM 推理，`agent_execution` + `execution_step` 是让整个执行过程可见、可审计、可监控的数据基础。

#### agent_execution（Agent 执行记录）

| 字段 | 说明 |
|---|---|
| id | 执行 ID |
| org_id | 所属组织 |
| agent_id | 哪个 Agent 执行 |
| user_id | 谁触发 |
| conversation_id | 关联会话（可为空，程序触发时无会话） |
| status | 执行状态（等待中 / 执行中 / 成功 / 失败） |
| total_tokens | 本次执行消耗的总 Token 数 |
| cost | 本次执行成本（单位：元/美元，精度到 0.000001） |
| error_message | 失败原因 |
| started_at | 开始时间 |
| finished_at | 结束时间 |
| created_at | 记录时间 |

#### execution_step（执行步骤）

一次 Agent 执行可能拆成多个步骤，每个步骤是一次具体的操作（LLM 调用、工具调用、规划、检索等），这里记录每一步的输入输出和耗时。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| execution_id | 所属执行记录 |
| step_index | 步骤顺序号 |
| step_type | 步骤类型（llm / tool_call / retrieval / planning） |
| tool_name | 工具名称（仅 tool_call 类型时有值） |
| input | 步骤输入（JSON） |
| output | 步骤输出（JSON） |
| status | 步骤状态（成功 / 失败） |
| tokens | 本步骤消耗 Token（LLM 类型时记录） |
| duration_ms | 执行耗时（毫秒） |
| created_at | 步骤记录时间 |

**为什么要这两张表：**

```
用户："帮我生成本周销售分析报告并发送给老板"

→ agent_execution（一条记录，状态：成功，tokens：2150，cost：0.003元）
    ↓
    execution_step 1 | planning    | 拆解任务为3步
    execution_step 2 | tool_call   | query_database → 返回本周销售数据
    execution_step 3 | tool_call   | generate_excel → 生成报表文件
    execution_step 4 | llm         | 撰写邮件正文
    execution_step 5 | tool_call   | send_email → 发送成功
```

有了这两张表，才能实现：Agent 执行监控面板、Token 消耗统计、成本分析、执行步骤调试（Debug）。

#### workflow（工作流定义）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| name | 工作流名称 |
| definition_json | 流程定义（节点、连线、触发条件，JSON 结构） |
| creator_id | 创建人 |
| status | 状态（启用 / 禁用） |
| created_at | 创建时间 |

#### workflow_task（工作流任务执行记录）

工作流本身是模板，`workflow_task` 才是每一次具体跑起来的实例，未来"AI 自动执行"的可观测性全部依赖这张表。

| 字段 | 说明 |
|---|---|
| id | 主键 |
| workflow_id | 所属工作流定义 |
| trigger_type | 触发方式（手动 / 定时 / 事件） |
| status | 执行状态（等待中 / 执行中 / 成功 / 失败） |
| input_data | 触发时的输入参数（JSON） |
| output_data | 执行结果（JSON） |
| error_message | 失败原因（失败时记录） |
| started_at | 开始时间 |
| finished_at | 结束时间 |

---

### 4.6 系统日志

#### system_log（系统日志）

| 字段 | 说明 |
|---|---|
| id | 主键 |
| org_id | 所属组织 |
| user_id | 操作人（系统自动触发时可为空） |
| action | 操作类型（如 `user.login`、`document.delete`） |
| target_type | 操作对象类型 |
| target_id | 操作对象 ID |
| detail | 详情（JSON） |
| ip_address | 操作来源 IP |
| created_at | 操作时间 |

Append-only 表，不提供 UPDATE / DELETE 接口，只允许按时间维度归档迁移到冷存储。

---

## 5. 表定义（Table Definition）

以核心表 `document_chunk` 和 `workflow_task` 为例给出完整 DDL，其余表按同一规范补齐。

```sql
CREATE TABLE document_chunk (
    id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    document_id     BIGINT UNSIGNED NOT NULL,
    chunk_index     INT UNSIGNED NOT NULL,
    content         TEXT NOT NULL,
    token_count     INT UNSIGNED NOT NULL DEFAULT 0,
    vector_id       VARCHAR(64) NOT NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (document_id) REFERENCES document(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE workflow_task (
    id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    workflow_id     BIGINT UNSIGNED NOT NULL,
    trigger_type    VARCHAR(16) NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'pending',
    input_data      JSON NULL,
    output_data     JSON NULL,
    error_message   VARCHAR(512) NULL,
    started_at      DATETIME NULL,
    finished_at     DATETIME NULL,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**统一规范：**

- 所有表使用 `InnoDB` 引擎、`utf8mb4` 字符集（兼容 Emoji 和多语言内容）
- 主键统一使用 `BIGINT UNSIGNED AUTO_INCREMENT`
- 所有业务表统一包含三个时间字段：`created_at`（创建时间）、`updated_at`（最后更新时间）、`deleted_at`（软删除时间，NULL 表示未删除）
- 软删除优先——不做物理删除，查询时默认过滤 `deleted_at IS NULL`；`system_log` 是唯一例外（Append-only，不提供删除接口）

---

## 6. 索引设计（Index Design）

索引设计的原则：覆盖高频查询路径 + 覆盖多租户隔离字段。几乎每张业务表的第一个索引都应该是 `org_id`，或者是"`org_id` 打头的复合索引"。

| 表 | 索引 | 用途 |
|---|---|---|
| user | `UNIQUE INDEX (org_id, username)` | 登录查询，且用户名只需在组织内唯一 |
| user | `INDEX (email)` | 找回密码、邮箱登录 |
| knowledge_base | `INDEX (org_id)` | 按组织列出知识库 |
| document | `INDEX (kb_id)` | 按知识库列出文档 |
| document | `INDEX (kb_id, status)` | 筛选某知识库下处理失败/待处理的文档 |
| document_chunk | `INDEX (document_id, chunk_index)` | 按文档还原切片顺序 |
| document_chunk | `UNIQUE INDEX (vector_id)` | 向量库命中后反查原文，要求唯一 |
| conversation | `INDEX (user_id, created_at)` | 用户会话列表按时间排序 |
| message | `INDEX (conversation_id, created_at)` | 会话内消息按时间拉取，这是最高频查询 |
| agent_tool | `UNIQUE INDEX (agent_id, tool_id)` | 防止重复绑定，同时加速按 Agent 查工具列表 |
| workflow_task | `INDEX (workflow_id, created_at)` | 查看某个工作流的历史执行记录 |
| workflow_task | `INDEX (status, created_at)` | 调度器扫描待执行/执行中的任务，这是任务系统最核心的查询 |
| agent_execution | `INDEX (org_id, agent_id, created_at)` | 按组织 + Agent 查执行历史 |
| agent_execution | `INDEX (user_id, created_at)` | 按用户查自己触发的执行记录 |
| agent_execution | `INDEX (status, created_at)` | 监控面板筛选执行中/失败的任务 |
| execution_step | `INDEX (execution_id, step_index)` | 按执行记录按序拉取所有步骤，最高频查询 |
| model_config | `INDEX (org_id, status)` | 按组织列出可用模型 |
| prompt_template | `INDEX (org_id, creator_id)` | 按组织和创建人筛选模板 |
| system_log | `INDEX (org_id, created_at)` | 按组织按时间查审计日志 |
| system_log | `INDEX (user_id, created_at)` | 按操作人查日志 |

**特别说明 `workflow_task (status, created_at)`：** 任务调度器需要持续扫描"状态为等待中"的任务，这是一个持续高频的轮询查询（或者由消息队列触发，但监控页面仍需要按状态筛选），`status` 放在复合索引最前面能让筛选直接命中索引，`created_at` 用于排序，避免额外的文件排序（filesort）。

---

## 7. 未来扩展（Future Extension）

以下方向在当前表结构设计时已预留扩展空间，不需要在 v1 阶段实现，但表结构不应阻碍它们后续加入：

**agent_version（Agent 配置版本管理）**
企业使用中 Prompt 一定会迭代。当前 `agent` 表直接存 `system_prompt`，意味着修改后历史执行记录就无法还原当时用的是哪个 Prompt。未来新增 `agent_version` 子表（字段：agent_id、version、system_prompt、model_id、temperature、tools_config、created_at），`agent` 表只保存"当前生效版本号"，历史版本在子表里完整保留。面试可以说：支持 Agent 配置版本管理，避免 Prompt 迭代导致历史任务不可追踪。

**workflow_node（Workflow 可视化节点）**
当前 `workflow.definition_json` 把整个流程存成一个 JSON 是 MVP 阶段的合理选择，能快速上线。未来要支持类似 Dify / n8n / LangGraph 的可视化编排，需要拆出 `workflow_node` 表（字段：workflow_id、node_type、config、position_x、position_y），节点之间的连线存在 `workflow_edge` 表。这个拆分不影响 `workflow` 主表结构，只是把 JSON 里的数据迁移到独立表并加索引。

**data_permission（数据权限）**
当前 RBAC 管控的是"能不能执行某个操作"（功能权限），但企业里还有"能看哪些数据"的需求（数据权限）：销售经理能看本部门所有销售记录，普通销售只能看自己的。未来在 `role` 上扩展一个 `data_scope` 字段（ALL / DEPARTMENT / SELF / CUSTOM），并为支持数据权限过滤的资源类型新增 `data_permission` 配置表，不需要改动现有 RBAC 层。

**Multi-Agent 协作**
`conversation.agent_id` 未来扩展为多对多，新增 `conversation_agent` 关联表，不影响现有结构。

**插件市场**
`tool.type` 已包含 MCP Server 类型，未来新增 `plugin_install` 表记录组织安装了哪些市场插件，与 `tool` 建立引用关系。

**审计日志归档**
`system_log` 数据量增长后，按月分表或归档到 ClickHouse，当前字段设计已支持直接迁移，无需转换。

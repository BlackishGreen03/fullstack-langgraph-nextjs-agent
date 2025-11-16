# AI Agent 使用说明

这是一个基于 Next.js 15 的全栈应用，使用 LangGraph.js 实现了 AI 智能体聊天界面，集成了模型上下文协议（MCP）服务器和流式响应功能。

## 架构概览

### 核心智能体系统

- **LangGraph 智能体**：使用 `src/lib/agent/builder.ts` 中的 `AgentBuilder` 类构建 - 创建 agent→tool_approval→tools 流程的状态图
- **MCP 集成**：从存储在 Postgres 中的 MCP 服务器动态加载工具（`src/lib/agent/mcp.ts`）
- **持久化内存**：使用 LangGraph 的 Postgres 检查点器实现跨会话的对话历史记录
- **工具审批**：实现人机交互模式，通过中断机制进行工具执行审批

### 数据流

1. 用户消息 → `/api/agent/stream` SSE 端点 → `agentService.ts` 中的 `streamResponse()`
2. 智能体使用启用的 MCP 服务器中的工具处理 → 流式增量响应
3. 前端使用带有 React Query 的 `useChatThread()` 钩子实现乐观 UI 和流式更新
4. 通过 Prisma → Postgres 实现线程持久化（线程 + MCP 服务器配置）

## 基本开发命令

```bash
# 设置（需要在 5434 端口运行 Postgres）
docker compose up -d
pnpm install
pnpm prisma:generate
pnpm prisma:migrate

# 开发
pnpm dev  # 使用 Turbopack 的 Next.js
pnpm prisma:studio  # 数据库 UI

# 数据库操作
pnpm prisma:generate  # 模式更改后
pnpm prisma:migrate   # 创建新迁移
```

## 项目特定模式

### 智能体配置

- **一次性设置**：`ensureAgent()` 确保在创建智能体之前初始化 Postgres 检查点器
- **动态工具加载**：每次创建智能体时从数据库查询 MCP 服务器
- **模型灵活性**：通过 `AgentConfigOptions` 支持在 OpenAI/Google 模型之间切换

### 流式架构

- **SSE 与 React Query**：`useChatThread` 管理乐观 UI + 流式更新
- **消息累积**：前端按消息 ID 连接文本块以实现流畅的用户体验
- **工具审批流程**：使用带有 `resume` 动作的 Command 对象而不是常规输入

### 数据库模式细节

- `MCPServer` 模型支持 stdio 和 http 两种 MCP 服务器类型，带有条件字段
- `Thread` 模型是最小的 - 实际的对话历史记录存储在 LangGraph 检查点中
- 使用 JSON 字段（`args`、`env`、`headers`）实现灵活的 MCP 服务器配置

### 组件结构

- **上下文提供器**：`ThreadContext` 管理活动线程，`UISettingsContext` 管理 UI 状态
- **自定义钩子**：`useChatThread`、`useMCPTools`、`useThreads` 处理特定数据域
- **消息组件**：AI/人类/工具/错误消息类型的独立组件，带有工具调用显示

### API 路由模式

- 流式端点使用 `dynamic = "force-dynamic"` 和 `runtime = "nodejs"`
- 流式查询参数：`content`、`threadId`、`model`、`allowTool`、`approveAllTools`
- MCP 服务器 CRUD 遵循 `/api/mcp-servers/route.ts` 中的 REST 模式

## 关键集成点

### MCP 服务器管理

- 通过 `MCPServerForm` 组件添加服务器 → 存储在数据库中 → 动态加载到智能体
- 服务器配置支持 stdio 类型的环境变量和命令参数
- 工具名称带有服务器名称前缀以防止冲突

### 工具审批工作流

- 智能体在工具调用时暂停，发出带有工具详情的中断
- 前端显示审批 UI，发送 allowTool=allow/deny 参数
- 使用 Command.resume() 模式而不是新消息输入

### 内存系统

- 通过 `getHistory(threadId)` 从 LangGraph 检查点检索线程历史
- 前端在流式传输期间乐观地更新 React Query 缓存
- Postgres 检查点器处理并发访问和持久化

修改智能体系统时，在模式更改后始终运行数据库迁移，并重新启动开发服务器以获取新的 MCP 服务器配置。

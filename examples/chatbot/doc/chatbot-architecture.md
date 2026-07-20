# Chatbot Example — 架构与前后端关联详解

## 一、Chatbot 项目是什么？

`examples/chatbot` 是 Spring AI Alibaba 的 **入门级示例项目**，展示如何用最少的代码搭建一个具备工具调用能力的 ReAct Agent，并通过内置的 Studio 前端界面与之对话。

核心能力：

- 使用 **ReAct Agent**（ReAct = Reasoning + Acting）模式，让 LLM 可以思考 → 行动 → 观察 → 再思考
- 挂载了 **3 个工具**：Shell 命令执行、Python 代码执行、文件内容查看
- 通过 **SSE（Server-Sent Events）** 流式返回对话结果
- 自带基于 Next.js 的 **Studio Web UI**，开箱即用

## 二、项目文件结构

```
examples/chatbot/
├── pom.xml                         # Maven 构建配置
└── src/main/java/com/alibaba/cloud/ai/examples/chatbot/
    ├── ChatbotApplication.java     # Spring Boot 启动入口
    ├── ChatbotAgent.java           # Agent 配置（ReActAgent + 工具注册）
    ├── AgentStaticLoader.java      # 将 Agent 注册到 Studio 的桥梁
    └── PythonTool.java             # 基于 GraalVM 的 Python 代码执行工具
```

项目没有 `application.yml` 或 `application.properties`——所有配置采用 Spring Boot 默认值和自动配置。

## 三、各 Java 类详解

### 3.1 ChatbotApplication.java — 启动入口

```
路径: src/main/java/com/alibaba/cloud/ai/examples/chatbot/ChatbotApplication.java
```

- 标准 Spring Boot 启动类，`@SpringBootApplication` 注解
- 唯一的定制是注册了一个 `ApplicationReadyEvent` 监听器，在应用启动完成后打印访问地址：

```
✅ Application is ready!
🚀 Chat with your agent: http://localhost:8080/chatui/index.html
```

### 3.2 ChatbotAgent.java — Agent 核心配置

```
路径: src/main/java/com/alibaba/cloud/ai/examples/chatbot/ChatbotAgent.java
```

这是整个项目的核心，通过 `@Configuration` 和多个 `@Bean` 定义了 Agent 及其工具链。共定义了 **5 个 Bean**：

| Bean | 类型 | 说明 |
|------|------|------|
| `chatbotReactAgent` | `ReactAgent` | 核心 Agent，组合了模型、指令、工具和记忆 |
| `memorySaver` | `MemorySaver` | 基于内存的检查点存储，支撑多轮对话的上下文记忆 |
| `executeShellCommand` | `ToolCallback` | Shell 命令执行工具（通过 `ShellTool` 封装） |
| `executePythonCode` | `ToolCallback` | Python 代码执行工具（通过 GraalVM Polyglot） |
| `viewTextFile` | `ToolCallback` | 文件查看工具（通过 `ReadFileTool` 封装） |

**ReActAgent 构建参数：**

```java
ReactAgent.builder()
    .name("SAA")                    // Agent 名称
    .model(chatModel)               // ChatModel（DashScope/OpenAI 等，由环境变量 AI_DASHSCOPE_API_KEY 决定）
    .instruction(INSTRUCTION)       // 系统指令
    .enableLogging(true)            // 开启日志
    .saver(memorySaver)             // 记忆存储
    .hooks(ShellToolAgentHook...)   // Shell 会话生命周期管理
    .tools(executeShellCommand, executePythonCode, viewTextFile)
    .build();
```

**系统指令 (INSTRUCTION)** 告诉 Agent 它的身份是 "SAA"，可以使用 Shell 命令、Python 代码和文本文件查看工具。

### 3.3 AgentStaticLoader.java — Agent 注册桥梁

```
路径: src/main/java/com/alibaba/cloud/ai/examples/chatbot/AgentStaticLoader.java
```

- 实现了 `AgentLoader` 接口（来自 `spring-ai-alibaba-studio` 模块）
- `@Component` 注解，Spring 自动检测并注册为 Bean
- 构造函数接收 `Agent` 注入（即 `ReactAgent`），编译其图结构，打印 PlantUML 表示
- 将 Agent 以 key `"research_agent"` 存入内部 map
- `listAgents()` 返回所有注册的 Agent 名称
- `loadAgent(name)` 按名称返回 Agent 实例

**这就是前后端关联的第一关键点**：用户如果定义了自己的 `AgentLoader` Bean，它就会取代 Studio 默认的 `ContextScanningAgentLoader`。这样前端通过 `GET /list-apps` 拿到的就是 `["research_agent"]`。

### 3.4 PythonTool.java — Python 执行工具

```
路径: src/main/java/com/alibaba/cloud/ai/examples/chatbot/PythonTool.java
```

- 实现了 `BiFunction<PythonRequest, ToolContext, String>`，是 Spring AI 工具回调的标准形式
- 基于 **GraalVM Polyglot API** 在 JVM 内执行 Python 代码
- **安全沙箱**：禁止 I/O、禁止创建进程、禁止 native 访问，只允许 host 对象访问
- 支持返回多种 Python 类型：字符串、数字、布尔值、数组/列表
- `PythonRequest` 内部类只包含一个 `code` 字段

## 四、pom.xml 依赖关系

```xml
<!-- 核心依赖 -->
spring-ai-alibaba-starter-dashscope     ← DashScope 灵积模型服务（LLM 提供商）
spring-ai-alibaba-agent-framework       ← Agent 框架（ReactAgent 等）
spring-ai-alibaba-studio                ← Studio 前端 + 后端控制器 ★ 前后端关联核心
spring-boot-starter                     ← Spring Boot 基础

<!-- 可选依赖 -->
polyglot + python-community (GraalVM)   ← 用于 PythonTool 在 JVM 内运行 Python
```

**关键**：`spring-ai-alibaba-studio` 这个依赖是整个前后端关联的根基——引入它，就自动获得了前端 UI 和后端 API。

## 五、前后端如何关联？

### 5.1 关联总览图

```
┌─────────────────────────────────────────────────────────────────┐
│                    chatbot (Spring Boot App)                    │
│                                                                 │
│  ┌──────────────────────┐     ┌──────────────────────────────┐  │
│  │  Java 后端（业务层）  │     │  spring-ai-alibaba-studio     │   │
│  │                      │     │                              │   │
│  │ ChatbotAgent ────────┼────→│ AgentStaticLoader            │   │
│  │ (ReactAgent Bean)    │     │ (implements AgentLoader)     │   │
│  │                      │     │         ↓                    │   │
│  │ MemorySaver          │     │   AgentController            │   │
│  │ Tools (Shell/Python/ │     │   ExecutionController        │   │
│  │        FileView)     │     │   ThreadController           │   │
│  │                      │     │                              │   │
│  └──────────────────────┘     │ ┌────────────────────────┐   │   │
│                               │ │ /chatui/** (静态资源)   │   │   │
│                               │ │  Next.js SPA           │   │   │
│                               │ │  index.html + JS/CSS   │   │   │
│                               │ └────────────────────────┘   │   │
│                               └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 关联层级拆解

**第 1 层：Maven 依赖引入**

chatbot 的 `pom.xml` 中直接依赖了 `spring-ai-alibaba-studio`。这意味着 Studio 模块的所有类、静态资源都会被打包进最终的 Spring Boot fat jar。

**第 2 层：Spring Boot 自动配置激活**

Studio 模块在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中注册了：

```
com.alibaba.cloud.ai.agent.studio.SaaStudioWebModuleAutoConfiguration
```

这个类只有两件事：
1. `@Configuration` 声明
2. `@ComponentScan(basePackages = "com.alibaba.cloud.ai.agent.studio")` 扫描整个 Studio 包

被扫描到的组件包括：
- **Controller 层**：`AgentController`、`ExecutionController`、`ThreadController`、`GraphController`、`ChatUiRedirectController`
- **Service 层**：`RunnerService`、`ThreadService`
- **Loader 层**：`StudioLoaderAutoConfiguration`（提供默认 `AgentLoader`、`GraphLoader`）
- **Config 层**：`WebConfig`（CORS 配置）

**第 3 层：AgentLoader — 前后端的数据桥梁**

这是理解前后端关联最关键的部分。流程如下：

1. `StudioLoaderAutoConfiguration` 注册默认的 `ContextScanningAgentLoader`（`@ConditionalOnMissingBean(AgentLoader.class)`）
2. chatbot 项目中，`AgentStaticLoader` 也是 `@Component` 且实现了 `AgentLoader`
3. 由于 chatbot 提供了自定义的 `AgentLoader` Bean，默认的 `ContextScanningAgentLoader` **不会被创建**（`@ConditionalOnMissingBean` 条件不满足）
4. 于是所有依赖 `AgentLoader` 的组件（AgentController、ExecutionController、RunnerService）都注入的是 `AgentStaticLoader`

```
AgentStaticLoader (chatbot 提供)
       ↓ 注入
AgentController    → GET  /list-apps          返回 ["research_agent"]
ExecutionController → POST /run_sse            加载 agent 执行对话
RunnerService       → getAgent(name)           辅助获取 agent
```

**第 4 层：前端静态资源服务**

Studio 模块将 Next.js 构建产物放在 `META-INF/resources/chatui/` 下：

```
META-INF/resources/chatui/
├── index.html              # Next.js 入口
├── _next/static/           # JS/CSS/font 资源
├── agent/                  # /agent/[agentName] 路由
└── graph/                  # /graph/[graphName] 路由
```

Spring Boot 会自动将 `META-INF/resources/` 下的内容映射为静态资源，因此访问 `http://localhost:8080/chatui/index.html` 就直接进入了前端 SPA。

`ChatUiRedirectController` 额外提供了 `/chatui` → `/chatui/index.html` 的重定向。

**第 5 层：前端 API 调用链**

```
前端 (Next.js SPA)                        后端 (Spring Boot)
══════════════════                        ═══════════════════

1. 页面加载
   GET /list-apps                  →   AgentController.listApps()
                                       → AgentStaticLoader.listAgents()
                                       ← ["research_agent"]

2. 创建会话
   POST /apps/research_agent/
        users/{userId}/
        threads/{threadId}         →   ThreadController.createThreadWithId()
                                       → ThreadService.createThread()
                                       ← Thread JSON

3. 发送消息（SSE 流式）
   POST /run_sse                   →   ExecutionController.agentRunSse()
   Content-Type: application/json      → AgentStaticLoader.loadAgent("research_agent")
   Accept: text/event-stream           → agent.stream(userMessage, runnableConfig)
                                        ← SSE: data: {agent, node, message, ...}
                                        ← SSE: data: {agent, node, message, ...}
                                        ← ...更多流式事件
```

### 5.3 SSE 事件格式

前端按 SSE 协议接收事件，每个 `data` 字段是一个 JSON 对象，结构为 `AgentRunResponse`：

```json
{
  "node": "agent",
  "agent": "SAA",
  "message": {
    "messageType": "assistant",
    "content": "Hello! How can I help you?"
  },
  "tokenUsage": { ... },
  "chunk": "Hello! How can I help you?"
}
```

消息类型通过 `messageType` 字段做多态区分：
- `"assistant"` — 普通助手回复文本
- `"tool-request"` — Agent 发起工具调用
- `"tool"` — 工具执行结果返回
- `"tool-confirm"` — 需要人工确认的工具调用（中断机制）
- `"user"` — 用户消息

## 六、完整运行流程

### 启动阶段

```
./mvnw -pl examples/chatbot spring-boot:run
      │
      ▼
Spring Boot 启动
      │
      ├─→ SaaStudioWebModuleAutoConfiguration 激活
      │     └─→ 注册所有 Controller / Service / Loader / WebConfig
      │
      ├─→ ChatbotAgent 创建 Bean:
      │     ├─ memorySaver (MemorySaver)
      │     ├─ executeShellCommand (ShellTool → ToolCallback)
      │     ├─ executePythonCode (PythonTool → ToolCallback)
      │     ├─ viewTextFile (ReadFileTool → ToolCallback)
      │     └─ chatbotReactAgent (ReactAgent)
      │
      ├─→ AgentStaticLoader 初始化:
      │     ├─ 注入 ReactAgent
      │     ├─ 编译 Agent 图 → 打印 PlantUML
      │     └─ 注册到内部 map: {"research_agent" → ReactAgent}
      │
      └─→ ApplicationReadyEvent 触发
            └─→ 打印: 🚀 Chat with your agent: http://localhost:8080/chatui/index.html
```

### 用户对话阶段

```
浏览器打开 http://localhost:8080/chatui/index.html
      │
      ▼
Next.js SPA 加载
      │
      ├─→ GET /list-apps → ["research_agent"]
      │     前端知道有一个叫 "research_agent" 的 agent 可用
      │
      ├─→ 用户创建 Thread
      │     POST /apps/research_agent/users/user123/threads/thread-001
      │
      ├─→ 用户输入 "列出当前目录的文件"
      │
      ▼
POST /run_sse
Body: {
  "appName": "research_agent",
  "userId": "user123",
  "threadId": "thread-001",
  "newMessage": { "messageType": "user", "content": "列出当前目录的文件" }
}
      │
      ▼
ExecutionController.agentRunSse()
      │
      ├─→ agentLoader.loadAgent("research_agent") → ReactAgent
      ├─→ RunnableConfig(threadId="thread-001")  ← 关联 MemorySaver 检查点
      ├─→ agent.stream(userMessage, runnableConfig)
      │
      ▼
ReactAgent 内部流程:
      │
      ├─→ LLM 调用 (DashScope/OpenAI)
      │     "用户想列目录，我有 execute_shell_command 工具..."
      │
      ├─→ 决定调用工具: execute_shell_command("ls -la")
      │     └─→ SSE 事件: messageType="tool-request"
      │
      ├─→ ShellTool 执行命令，返回结果
      │     └─→ SSE 事件: messageType="tool"
      │
      ├─→ LLM 再次调用，基于工具结果生成回复
      │     "当前目录有以下文件: ..."
      │     └─→ SSE 事件: messageType="assistant" (流式逐 chunk)
      │
      └─→ 流结束
```

## 七、3 个工具详解

| 工具名称 | 实现类 | 底层技术 | 功能 |
|---------|--------|---------|------|
| `execute_shell_command` | `ShellTool`（框架内置） | Java `ProcessBuilder`，持久化工作目录 | 在 `/tmp/agent-workspace` 中执行 Shell 命令，支持 `&&` 链式调用、超时控制、输出截断 |
| `execute_python_code` | `PythonTool`（自定义） | GraalVM Polyglot + Python Community | 在 JVM 沙箱中执行 Python 代码片段，禁止 I/O 和进程创建 |
| `view_text_file` | `ReadFileTool`（框架内置） | Java NIO 文件读取 | 读取文本文件内容，支持 offset/limit 分页，默认最多 500 行 |

> **注意**：`executeShellCommand` 必须配合 `ShellToolAgentHook` 使用，该 Hook 负责 Shell 会话的创建和生命周期管理。

## 八、关键设计决策与值得注意的点

### 8.1 为什么需要 AgentStaticLoader？

Studio 默认的 `ContextScanningAgentLoader` 会扫描所有 `Agent` 类型的 Bean 并用 `Agent.name()` 作为 key。chatbot 使用自定义的 `AgentStaticLoader` 有两个目的：

1. **控制 Agent 在前端的显示名称**：内部 map 的 key 是 `"research_agent"`，而不是 Agent 的默认名称 `"SAA"`
2. **在初始化时编译图并打印 PlantUML**：方便开发者可视化 Agent 的执行流

### 8.2 MemorySaver 的作用

`MemorySaver` 是 graph-core 提供的**检查点存储**实现，基于内存的 `ConcurrentHashMap`。它让 Agent 支持：

- **多轮对话**：每次对话的状态（已完成的节点、中间结果）被保存，下次同一 threadId 的请求可以从中断点继续
- **中断恢复**：当 Agent 遇到需要人工确认的工具调用时，状态被保存，用户确认后 Agent 从断点恢复执行
- **生产环境中**可以替换为 PostgreSQL/MySQL/Redis 等持久化实现

### 8.3 SSE 流式推送的过滤

`ExecutionController.executeAgent()` 中有一个关键的 filter：

```java
.filter(nodeOutput -> !(nodeOutput instanceof StreamingOutput<?> so
        && so.getOutputType() == OutputType.AGENT_MODEL_FINISHED))
```

这过滤掉了 `AGENT_MODEL_FINISHED` 事件——它是框架对 `AGENT_MODEL_STREAMING` chunks 的聚合，如果不过滤会导致前端收到重复内容。

### 8.4 前端路由说明

Next.js 前端有两个主要路由：

- `/agent/[agentName]` — Agent 对话界面（chatbot 的核心使用场景）
- `/graph/[graphName]` — 图工作流调试界面（本项目未使用）

静态文件中的 `agent/__.html` 和 `agent/__.txt` 是 Next.js 静态导出的占位符，Spring Boot 的 `ChatUiRedirectController` 将 `/chatui` 重定向到 `/chatui/index.html`。

## 九、如何运行

```shell
# 1. 设置 LLM API Key
export AI_DASHSCOPE_API_KEY=your-api-key

# 2. 从项目根目录启动
./mvnw -pl examples/chatbot spring-boot:run

# 3. 打开浏览器
open http://localhost:8080/chatui/index.html
```

启动后，可以在 Studio UI 中与 Agent 对话，例如：

- "列出当前目录下的文件"
- "用 Python 计算 1 到 100 的和"
- "查看 /tmp/agent-workspace/ 下的某个文件内容"

Agent 会自动选择合适的工具来执行任务。

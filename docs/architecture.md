# yu-ai-agent 项目架构文档

## 技术栈

| 层次 | 技术 |
|------|------|
| 框架 | Spring Boot 3.4.4 + Java 21 |
| AI 框架 | Spring AI 1.0.0 + Spring AI Alibaba 1.0.0.2 |
| 大模型 | 阿里云 DashScope（通义千问）+ Ollama（本地模型） |
| 向量存储 | SimpleVectorStore / PgVector（PostgreSQL） |
| 前端 | 独立前端项目（yu-ai-agent-frontend） |
| MCP 服务 | 独立子模块（yu-image-search-mcp-server） |

---

## 整体分层架构

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                   Controller 层                          │
│   AiController  ──  HealthController                    │
│   /ai/love_app/chat/*   /ai/manus/chat                  │
└─────────────┬──────────────────┬───────────────────────┘
              │                  │
              ▼                  ▼
┌─────────────────────┐  ┌──────────────────────────────┐
│       App 层         │  │          Agent 层             │
│     LoveApp         │  │  BaseAgent（状态机+执行循环） │
│  （结构化 AI 应用）  │  │  ReActAgent（think-act 模式）│
│   - doChat          │  │  ToolCallAgent（工具调用实现）│
│   - doChatWithRag   │  │  YuManus（超级智能体实例）    │
│   - doChatWithTools │  └──────────────────────────────┘
│   - doChatWithMcp   │
└─────────────────────┘
              │                  │
              └──────┬───────────┘
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Spring AI ChatClient 层                     │
│  ChatClient.prompt().user().advisors().tools().call()   │
└───────────┬─────────────────────────────────────────────┘
       ┌────┴────┐
       ▼         ▼
┌───────────┐  ┌──────────────────────────────────────────┐
│ Advisor层  │  │                Tool 层                   │
│ (拦截器链) │  │  FileOperationTool   WebSearchTool       │
│ MyLogger  │  │  WebScrapingTool     TerminalTool        │
│ ReReading │  │  ResourceDownload    PDFGenerationTool   │
│ ChatMemory│  │  TerminateTool（触发 Agent 终止状态）     │
│ RAGAdvisor│  └──────────────────────────────────────────┘
└───────────┘
```

---

## Agent 模块 —— 核心继承体系

采用**模板方法模式**三层继承，是整个项目最重要的设计。

```
BaseAgent（抽象类）
│
├── 职责：状态机管理 + 执行循环控制
├── 状态机：IDLE → RUNNING → FINISHED / ERROR
├── 维护 messageList（手动管理多轮消息上下文）
├── run()       同步执行，最多 maxSteps 步
├── runStream() 异步执行，SSE 流式输出每步结果
└── abstract step()  ← 子类定义单步逻辑
         │
         ▼
    ReActAgent（抽象类）
    │
    ├── 职责：实现 ReAct 思维范式（Reasoning + Acting）
    ├── step() = think() + act()
    ├── abstract think() → boolean（是否需要执行工具）
    └── abstract act()   → String（工具执行结果）
              │
              ▼
         ToolCallAgent（具体类）
         │
         ├── 职责：实现具体的工具调用逻辑
         ├── think()：调用 LLM，解析返回的工具调用列表
         │           无工具调用 → 返回 false（结束循环）
         │           有工具调用 → 返回 true（继续执行）
         ├── act()：  通过 ToolCallingManager 批量执行工具
         │           检测到 TerminateTool → 设置 state=FINISHED
         └── 禁用 Spring AI 内置工具执行（自维护消息上下文）
                   │
                   ▼
              YuManus（Spring Bean）
              │
              ├── 注入 allTools[]（7 个工具）
              ├── systemPrompt：定义 Agent 角色和能力
              ├── nextStepPrompt：每步引导 LLM 选择工具
              └── maxSteps = 20
```

### 关键设计决策

> `ToolCallAgent` 通过 `DashScopeChatOptions.withInternalToolExecutionEnabled(false)` 禁用了 Spring AI 内置的工具执行机制，转而手动管理消息上下文。
>
> **原因**：Spring AI 内置机制会在一次 `call()` 内自动完成工具执行，无法实现跨步骤的状态持久化和精细控制。自己维护 `messageList` 才能实现真正的多步骤 Agent 循环。

---

## LoveApp —— AI 应用能力矩阵

`LoveApp` 不是 Agent，是一个演示 Spring AI 各种能力的结构化 AI 应用。

| 方法 | 能力 | 关键 API |
|------|------|---------|
| `doChat()` | 基础多轮对话 | `MessageWindowChatMemory` |
| `doChatByStream()` | SSE 流式输出 | `.stream().content()` |
| `doChatWithReport()` | 结构化输出 | `.call().entity(Class)` |
| `doChatWithRag()` | RAG 知识库问答 | `QuestionAnswerAdvisor` + `QueryRewriter` |
| `doChatWithTools()` | 工具调用 | `.toolCallbacks(allTools)` |
| `doChatWithMcp()` | MCP 服务调用 | `ToolCallbackProvider` |

---

## Tool 层 —— 7 个内置工具

| 工具类 | 功能 |
|--------|------|
| `FileOperationTool` | 文件读写、创建、删除 |
| `WebSearchTool` | 调用 SearchAPI 进行网络搜索 |
| `WebScrapingTool` | 抓取网页内容 |
| `ResourceDownloadTool` | 下载网络资源到本地 |
| `TerminalOperationTool` | 执行终端命令 |
| `PDFGenerationTool` | 生成 PDF 文件 |
| `TerminateTool` | **终止 Agent 执行**（特殊工具） |

所有工具通过 `ToolRegistration` 配置类统一注册为 Spring Bean（`ToolCallback[]`）。

---

## Advisor 拦截器链

类似 Spring MVC 拦截器，在每次 LLM 调用前后执行，支持同步和流式两种模式。

```
请求 → [MyLoggerAdvisor] → [ReReadingAdvisor] → [ChatMemoryAdvisor] → [RAGAdvisor] → LLM
响应 ← [MyLoggerAdvisor] ← [ReReadingAdvisor] ← [ChatMemoryAdvisor] ← [RAGAdvisor] ← LLM
```

| Advisor | 作用 |
|---------|------|
| `MyLoggerAdvisor` | 打印请求/响应日志（info 级别） |
| `ReReadingAdvisor` | RE2 推理增强，重复读取问题提升推理质量 |
| `MessageChatMemoryAdvisor` | 注入历史消息，实现多轮对话记忆 |
| `QuestionAnswerAdvisor` | RAG 检索增强，从向量库检索相关文档 |

---

## RAG 模块

```
文档加载 → 文本切割 → 关键词增强 → 向量化 → 存储
  │           │           │           │        │
LoveApp   MyToken     MyKeyword   Embedding  VectorStore
Document  TextSplitter  Enricher   Model    (Simple/PgVector)
Loader

查询时：
用户问题 → QueryRewriter（LLM改写） → 向量检索 → 注入上下文 → LLM回答
```

| 类 | 职责 |
|----|------|
| `LoveAppDocumentLoader` | 加载 Markdown 格式知识库文档 |
| `MyTokenTextSplitter` | 按 Token 数量切割文档 |
| `MyKeywordEnricher` | 为文档片段添加关键词元数据 |
| `QueryRewriter` | 用 LLM 对用户查询进行语义改写 |
| `LoveAppVectorStoreConfig` | 配置本地 SimpleVectorStore |
| `PgVectorVectorStoreConfig` | 配置 PostgreSQL 向量存储 |
| `LoveAppRagCustomAdvisorFactory` | 自定义 RAG Advisor（文档查询+上下文增强） |

---

## MCP 子模块

`yu-image-search-mcp-server` 是一个独立的 MCP（Model Context Protocol）服务器，提供图片搜索工具，可以被主服务通过 MCP 协议远程调用。支持两种传输模式：
- **stdio 模式**：标准输入输出，适合本地调用
- **SSE 模式**：HTTP + Server-Sent Events，适合远程调用

---

## API 接口一览

| 接口 | 方法 | 说明 |
|------|------|------|
| `/ai/love_app/chat/sync` | GET | 同步调用恋爱大师 |
| `/ai/love_app/chat/sse` | GET | SSE 流式调用（Flux） |
| `/ai/love_app/chat/server_sent_event` | GET | ServerSentEvent 包装 |
| `/ai/love_app/chat/sse_emitter` | GET | SseEmitter 方式 |
| `/ai/manus/chat` | GET | 调用 YuManus 超级智能体（SSE） |

---

## 启动指南

### 必填配置

打开 `src/main/resources/application.yml`，替换以下两个 key：

```yaml
spring:
  ai:
    dashscope:
      api-key: your-api-key   # 阿里云百炼平台申请

search-api:
  api-key: 你的 API Key       # searchapi.io 申请，YuManus 网络搜索用
```

### 启动后端

用 IDE 运行 `YuAiAgentApplication.java`，或命令行：

```bash
./mvnw spring-boot:run
```

启动后访问接口文档：`http://localhost:8123/api/doc.html`

### 可选功能

| 功能 | 依赖 | 开启方式 |
|------|------|---------|
| PgVector 向量存储 | PostgreSQL + pgvector 扩展 | 取消 `datasource` 和 `vectorstore.pgvector` 注释 |
| MCP 图片搜索 | 先启动 `yu-image-search-mcp-server` | 取消 `ai.mcp.client` 注释 |
| Ollama 本地模型 | 本地安装并运行 Ollama | 已有配置，直接注入 `ollamaChatModel` Bean |

---

## 学习路线

> 共分 4 个阶段，由浅入深，每个阶段都有明确的读码目标和需要理解的核心问题。

---

### 阶段一：理解 Spring AI 基础调用（热身）

**目标**：搞清楚 Spring AI 是如何调用大模型的，理解 `ChatModel` 和 `ChatClient` 的关系。

**读码顺序**：

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `demo/invoke/SpringAiAiInvoke.java` | 最简单的调用：`ChatModel.call(Prompt)` |
| 2 | `demo/invoke/HttpAiInvoke.java` | 直接 HTTP 调用，理解底层本质 |
| 3 | `demo/invoke/SdkAiInvoke.java` | 用阿里 SDK 直接调用，不经过 Spring AI |
| 4 | `app/LoveApp.java` 的 `doChat()` | `ChatClient` 封装了什么，和 `ChatModel` 有何区别 |

**需要理解的问题**：
- `ChatModel` vs `ChatClient`：前者是底层模型接口，后者是带链式调用的高级封装
- `Prompt` 是什么结构：`systemMessage` + `userMessage` + `ChatOptions`
- 为什么 `doChatByStream()` 返回 `Flux<String>` 而不是 `String`

---

### 阶段二：掌握 Spring AI 核心能力（进阶）

**目标**：理解 Spring AI 的四大核心能力：记忆、结构化输出、RAG、工具调用。

#### 2.1 多轮对话记忆

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `LoveApp.java` 的 `doChat()` | `MessageChatMemoryAdvisor` 如何注入历史消息 |
| 2 | `chatmemory/FileBasedChatMemory.java` | 自己实现 `ChatMemory` 接口，用 Kryo 序列化消息到文件 |

**需要理解的问题**：
- `chatId` 是什么：会话隔离的 key，不同 chatId 对应不同的历史消息
- 内存记忆 vs 文件记忆的取舍：重启后是否需要保留历史
- `MessageWindowChatMemory` 的窗口大小（maxMessages=20）控制什么

#### 2.2 结构化输出

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `LoveApp.java` 的 `doChatWithReport()` | `.call().entity(LoveReport.class)` 如何把 AI 回复转成 Java 对象 |

**需要理解的问题**：
- Spring AI 内部用什么机制把 JSON 字符串映射成 Java Record
- 为什么用 `record LoveReport(...)` 而不是普通 class

#### 2.3 Advisor 拦截器链

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `advisor/MyLoggerAdvisor.java` | 同时实现 `CallAdvisor` 和 `StreamAdvisor`，`before()`/`observeAfter()` 分别做了什么 |
| 2 | `advisor/ReReadingAdvisor.java` | 如何在 `before()` 里改写 Prompt（RE2 技术） |

**需要理解的问题**：
- Advisor 的执行顺序由 `getOrder()` 控制，数字越小越先执行
- `adviseCall` vs `adviseStream`：同步调用和流式调用的拦截方式不同
- `ChatClientRequest.context()` 是什么：在 Advisor 间传递的上下文 Map

#### 2.4 RAG 知识库问答

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `rag/LoveAppDocumentLoader.java` | 如何加载 Markdown 文档 |
| 2 | `rag/MyTokenTextSplitter.java` | 为什么要切割文档，切太大/太小各有什么问题 |
| 3 | `rag/MyKeywordEnricher.java` | 给文档片段加元数据有什么用（后面可按 metadata 过滤） |
| 4 | `rag/LoveAppVectorStoreConfig.java` | 向量存储如何配置 |
| 5 | `rag/QueryRewriter.java` | 为什么要改写用户查询（原始查询不精准） |
| 6 | `LoveApp.java` 的 `doChatWithRag()` | 完整 RAG 链路：查询改写 → 向量检索 → 上下文注入 → LLM 回答 |
| 7 | `rag/LoveAppRagCustomAdvisorFactory.java` | 自定义 RAG：加相似度阈值、topK、元数据过滤 |

**需要理解的问题**：
- 向量化的本质：把文本变成高维向量，语义相近的向量距离更近
- `similarityThreshold(0.5)` 的作用：过滤掉相关性太低的文档
- `topK(3)` 的作用：只取最相关的 3 条文档注入上下文
- 查询改写的价值："失恋了怎么办" → 改写为更精准的语义查询

#### 2.5 工具调用

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `tools/WebSearchTool.java` | `@Tool` 注解的 description 是给 LLM 看的，LLM 根据它决定是否调用 |
| 2 | `tools/TerminateTool.java` | 最简单的工具，只有一个方法，没有参数 |
| 3 | `tools/ToolRegistration.java` | 如何用 `ToolCallbacks.from()` 批量注册工具 |
| 4 | `LoveApp.java` 的 `doChatWithTools()` | `.toolCallbacks(allTools)` 挂载工具后，Spring AI 自动完成调用 |

**需要理解的问题**：
- `@Tool(description=...)` 的 description 写给谁看：LLM 读这段描述来决定要不要调用这个工具
- `@ToolParam(description=...)` 的作用：告诉 LLM 这个参数是什么意思，怎么填
- Spring AI 内置工具调用流程：LLM 返回 tool_call → 框架自动执行 → 把结果再塞回上下文 → LLM 生成最终回答

---

### 阶段三：深入 Agent 核心（核心）

**目标**：彻底理解 ReAct Agent 的设计思想和实现细节，能自己写一个 Agent。

#### 3.1 状态机设计

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `agent/model/AgentState.java` | 4 个状态：IDLE / RUNNING / FINISHED / ERROR |
| 2 | `agent/BaseAgent.java` | `run()` 方法的完整流程：状态校验 → 执行循环 → 清理 |

**手绘状态转换图**（建议动手画）：
```
IDLE ──run()──▶ RUNNING ──正常完成──▶ FINISHED
                  │
                  ├── 达到 maxSteps ──▶ FINISHED
                  ├── 调用 TerminateTool ──▶ FINISHED
                  └── 抛出异常 ──▶ ERROR
```

**需要理解的问题**：
- 为什么 `run()` 开头要检查 `state != IDLE`：防止并发重入，一个 Agent 实例不能同时处理两个请求
- `finally` 块调用 `cleanup()` 的目的：无论成功还是失败都要重置状态
- `runStream()` 为什么要用 `CompletableFuture.runAsync()`：SSE 要求异步处理，不能阻塞主线程

#### 3.2 ReAct 思维范式

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `agent/ReActAgent.java` | `step()` = `think()` + `act()` 的完整实现 |

**ReAct 执行流程**：
```
用户输入
   │
   ▼
think() ──▶ 调用 LLM，LLM 思考：要不要用工具？用哪个？
   │
   ├── 不需要工具 ──▶ 直接返回结果，step 结束
   │
   └── 需要工具 ──▶ act() ──▶ 执行工具 ──▶ 结果写入 messageList ──▶ 下一个 step
```

**需要理解的问题**：
- `think()` 返回 `boolean` 的语义：`true` = 需要执行工具，`false` = 可以结束了
- 每个 step 之后 messageList 都增长了什么：`UserMessage(nextStepPrompt)` + `AssistantMessage` + `ToolResponseMessage`

#### 3.3 ToolCallAgent 的工具调用实现

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `agent/ToolCallAgent.java` | `think()` 和 `act()` 的完整实现 |

**think() 详细流程**：
```java
// 1. 把 nextStepPrompt 加入消息列表（引导 LLM 选择工具）
messageList.add(new UserMessage(nextStepPrompt));

// 2. 带着所有历史消息和工具列表调用 LLM
ChatResponse response = chatClient.prompt(new Prompt(messageList, chatOptions))
    .system(systemPrompt)
    .tools(availableTools)   // 告诉 LLM 有哪些工具可用
    .call().chatResponse();

// 3. 解析 LLM 的决策
List<ToolCall> toolCalls = response.getResult().getOutput().getToolCalls();
if (toolCalls.isEmpty()) return false;  // LLM 决定不用工具
else return true;                        // LLM 决定用工具
```

**act() 详细流程**：
```java
// 1. 通过 ToolCallingManager 执行所有工具
ToolExecutionResult result = toolCallingManager.executeToolCalls(prompt, toolCallChatResponse);

// 2. 更新消息列表（包含 AssistantMessage + ToolResponseMessage）
setMessageList(result.conversationHistory());

// 3. 检查是否调用了 TerminateTool
boolean terminated = toolResponseMessage.getResponses().stream()
    .anyMatch(r -> r.name().equals("doTerminate"));
if (terminated) setState(AgentState.FINISHED);  // 触发退出循环
```

**需要理解的问题**：
- 为什么要禁用 `withInternalToolExecutionEnabled(false)`：Spring AI 内置机制在一次 `call()` 内完成所有工具调用，无法实现跨步骤的持久化记忆
- `ToolCallingManager` 做了什么：把 LLM 返回的工具调用请求，映射到实际的 Java 方法并执行
- TerminateTool 的方法名必须是 `doTerminate`：`act()` 里硬编码了 `response.name().equals("doTerminate")`

#### 3.4 YuManus 的组装

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `agent/YuManus.java` | 如何配置 systemPrompt、nextStepPrompt，为什么这两个 Prompt 要分开 |

**两个 Prompt 的分工**：
- `systemPrompt`：整个会话固定不变，定义 Agent 的身份和能力边界
- `nextStepPrompt`：每一步都插入，引导 LLM 在当前这步做出正确的工具选择

---

### 阶段四：扩展能力（提高）

**目标**：理解 MCP 协议和 SSE 流式输出，能扩展 Agent 的工具集。

#### 4.1 MCP 工具集成

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `yu-image-search-mcp-server/tools/ImageSearchTool.java` | MCP Server 侧如何用 `@Tool` 暴露工具 |
| 2 | `yu-image-search-mcp-server/resources/application.yml` | MCP Server 的传输模式配置（stdio / SSE） |
| 3 | `LoveApp.java` 的 `doChatWithMcp()` | MCP Client 侧如何用 `ToolCallbackProvider` 接入远程工具 |

#### 4.2 SSE 流式输出

| 步骤 | 文件 | 重点关注 |
|------|------|---------|
| 1 | `AiController.java` | 4 种 SSE 实现方式的对比（Flux、ServerSentEvent、SseEmitter） |
| 2 | `BaseAgent.java` 的 `runStream()` | Agent 如何把每步结果实时推送到前端 |

**三种 SSE 方式对比**：

| 方式 | 适用场景 | 特点 |
|------|---------|------|
| `Flux<String>` | Spring WebFlux 项目 | 最简洁，响应式 |
| `Flux<ServerSentEvent<String>>` | 需要自定义 SSE 字段时 | 可设置 event/id/retry |
| `SseEmitter` | Spring MVC 项目 | 命令式编程风格，手动控制发送 |

---

### 学习检验：自己动手

完成以上学习后，尝试独立完成以下任务：

1. **新增一个工具**：写一个 `WeatherTool`，调用任意天气 API，注册到 `ToolRegistration` 里，用 YuManus 测试能否调用
2. **修改 Agent 行为**：把 YuManus 的 `maxSteps` 改成 5，观察 Agent 提前结束时的行为
3. **自定义 Advisor**：写一个统计每次请求耗时的 `TimingAdvisor`，加到 LoveApp 里
4. **实现持久化记忆**：把 LoveApp 的 `InMemoryChatMemoryRepository` 替换成 `FileBasedChatMemory`，重启服务后验证历史消息是否保留

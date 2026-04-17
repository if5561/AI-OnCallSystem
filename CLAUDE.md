# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 常用命令

### 核心开发
- `mvn clean install` —— 构建 Spring Boot 应用并执行 Maven 生命周期检查。
- `mvn spring-boot:run` —— 本地启动应用。
- `mvn test` —— 运行测试套件。当前仓库没有 `src/test` 目录，因此在新增测试前这条命令可能不会执行任何测试。
- `mvn -Dtest=ClassName test` —— 运行单个测试类。
- `mvn -Dtest=ClassName#methodName test` —— 运行单个测试方法。

### 本地环境 / Milvus
- `docker-compose -f vector-database.yml up -d` —— 启动本地 Milvus 依赖栈。
- `docker-compose -f vector-database.yml down` —— 停止本地 Milvus 依赖栈。
- `make init` —— 完整初始化本地环境：启动 Milvus、后台启动 Spring Boot、等待健康检查通过，然后将 `aiops-docs/*.md` 上传到向量库。
- `make up` / `make down` / `make status` —— 管理 Milvus 的 Docker 服务。
- `make start` / `make stop` / `make restart` —— 管理由 Makefile 启动的 Spring Boot 进程（使用 `server.pid`、`server.log`）。
- `make upload` —— 上传并索引 `aiops-docs/` 下的全部 Markdown 文档。
- `make test-upload` —— 上传示例文档 `aiops-docs/cpu_high_usage.md`。
- `make check` —— 通过 `GET /milvus/health` 检查应用健康状态。

## 运行配置

主要运行时配置位于 `src/main/resources/application.yml`。

关键配置项：
- 服务端口：`9900`
- 上传目录：`./uploads`
- Milvus 连接：`milvus.*`
- DashScope 聊天与向量化密钥：`spring.ai.dashscope.api-key`、`dashscope.api.key`
- RAG 检索条数：`rag.top-k`
- Prometheus 集成：`prometheus.*`
- CLS mock 开关：`cls.mock-enabled`
- Spring AI MCP 客户端 SSE 配置：`spring.ai.mcp.client.sse.connections.tencent-cls`

优先通过环境变量设置 `DASHSCOPE_API_KEY`，不要依赖 `application.yml` 中的兜底值。

## 架构概览

这是一个基于 Spring Boot 3.2 / Java 17 的应用，主要包含两条能力主线：
1. 基于 Spring AI Alibaba `ReactAgent` 的工具调用式聊天助手
2. 基于 DashScope + Milvus 的 RAG 与 AIOps 工作流

### 请求流转
- `src/main/java/org/example/Main.java` 启动 Spring Boot 应用。
- `src/main/java/org/example/controller/ChatController.java` 是主要 API 入口，暴露：
  - `/api/chat`：非流式对话
  - `/api/chat_stream`：SSE 流式对话
  - `/api/ai_ops`：SSE 流式 AIOps 分析
  - 会话历史清理与查询相关接口
- 聊天会话状态保存在 `ChatController` 的内存 `ConcurrentHashMap` 中，不做持久化。

### Chat Agent 组成
- `src/main/java/org/example/service/ChatService.java` 统一负责 Agent 装配。
- 它会创建 DashScope API 客户端与 `DashScopeChatModel`，基于内存中的历史消息拼接系统提示词，并构建 `ReactAgent`。
- Agent 工具来自两部分：
  - 本地 method tools：`DateTimeTools`、`InternalDocsTools`、`QueryMetricsTools`，以及可选的 `QueryLogsTools`
  - 通过 `ToolCallbackProvider` 注入的 MCP 工具
- `QueryLogsTools` 被设计为可选：如果没有注册本地实现，代码默认日志查询能力由 MCP 服务提供。

### AIOps 多 Agent 流程
- `src/main/java/org/example/service/AiOpsService.java` 实现 AIOps 主流程。
- 它构建一个 `SupervisorAgent` 来协调两个 `ReactAgent`：
  - `planner_agent`：负责规划 / 再规划下一步排查动作
  - `executor_agent`：只执行 planner 给出的第一步，并返回结构化证据
- Planner / Executor 循环的目标是产出 Markdown 告警分析报告，而不是 JSON。
- 这一流程复用了本地工具和 MCP 工具，因此 Prometheus、内部文档、日志查询能力都会进入告警分析链路。

### RAG 与文档索引链路
- 文件上传入口：`src/main/java/org/example/controller/FileUploadController.java`
- 文件上传成功后，会自动调用 `VectorIndexService.indexSingleFile(...)` 进行索引。
- 建库链路：
  1. `DocumentChunkService` 按标题、段落边界优先切分 Markdown / 文本文档。
  2. `VectorEmbeddingService` 调用 DashScope 生成向量。
  3. `VectorIndexService` 将分片内容和元数据写入 Milvus；若同一源文件已存在旧数据，会先删除再重建。
- 检索链路：
  1. `InternalDocsTools.queryInternalDocs(...)` 作为 Agent 工具暴露给模型。
  2. 它内部调用 `VectorSearchService.searchSimilarDocuments(...)`。
  3. `VectorSearchService` 先将查询文本向量化，再到 Milvus 做相似度检索。
- `RagService` 额外提供一条独立的“检索 + 生成”流式回答链路，会把检索结果与历史消息一起送入模型。

### 前端与静态资源
- `src/main/resources/static/` 下放着内置 Web 界面（`index.html`、`app.js`、`styles.css`）。
- 这些静态资源由 Spring Boot 直接对外提供。

### Milvus 集成
- `src/main/java/org/example/controller/MilvusCheckController.java` 提供 `GET /milvus/health` 健康检查接口。
- 本地 Milvus 依赖栈定义在 `vector-database.yml` 中，是文档索引和向量检索的基础依赖。

## 仓库特有说明
- 当前仅支持上传 `txt` 和 `md` 文件。
- Makefile 默认假设运行环境具备类 Unix shell 能力（如 `nohup`、`curl`、`tail`、`pkill`），即使仓库也可能在 Windows 环境中使用。
- `make upload` 依赖 `aiops-docs/` 目录下的 Markdown 文档，这也是推荐的本地初始化流程一部分。
- 当前仓库中没有现成的仓库级规则文件，如 `.cursorrules`、`.cursor/rules/` 或 `.github/copilot-instructions.md`。

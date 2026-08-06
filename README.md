# Ragent AI - Java Agentic RAG 实践

基于 Java 与 Spring Boot 构建的 Agentic RAG 应用，覆盖文档入库、问题理解、多路检索、MCP 工具调用、模型路由、流式回答和链路追踪。

> 本仓库基于开源项目 [nageoffer/ragent](https://github.com/nageoffer/ragent) 进行学习、部署与二次开发，遵循 Apache License 2.0。完整的原始功能说明和设计文档请参考[上游项目文档](https://nageoffer.com/ragent)。

![Ragent AI](assets/ragent-ai-banner.png)

## 项目目标

这个仓库用于实践 Java AI 应用的完整工程链路，而不仅是调用一次模型 API。当前重点包括：

- 理解问题重写、子问题拆分、意图路由和低置信度澄清；
- 掌握向量、关键词、图谱和联网搜索的多路召回，以及去重、RRF 融合与 Rerank；
- 理解 MCP 工具发现、参数提取、Schema 校验和异常降级；
- 实践 Redis 公平排队、分布式并发控制和 SSE 流式响应；
- 通过 Trace、回答溯源、用户反馈和评测接口观察 RAG 链路效果。

## 核心链路

```text
用户问题
  -> 加载会话记忆
  -> 问题重写与子问题拆分
  -> 意图识别与歧义判断
  -> 知识库检索 / MCP 工具调用
  -> 多路召回、去重、RRF、Rerank
  -> 上下文与引用组装
  -> LLM 流式生成
  -> SSE 返回正文、来源和推荐问题
```

主流程入口位于：

- `RAGChatController`：接收 SSE 问答请求；
- `RAGChatServiceImpl`：排队、Trace 和任务生命周期管理；
- `StreamChatPipeline`：编排重写、意图、检索与生成；
- `RetrievalEngine`：组织知识库检索与 MCP 工具调用；
- `MultiChannelRetrievalEngine`：并行执行检索通道和后处理链。

![RAG 核心链路](assets/ragent-chain-v3.png)

## 关键能力

### 问题理解与路由

- 使用历史消息补全多轮对话中的指代信息；
- 将复合问题拆成多个可独立检索的子问题；
- 通过树形意图识别选择知识库、MCP 或系统回答；
- 对候选意图接近的问题返回澄清选项，而不是直接猜测。

### 多路检索

- 支持向量、Elasticsearch 关键词、LightRAG 图谱和联网搜索通道；
- 检索通道通过专用线程池并行执行，单通道异常不会中断主链路；
- 后处理链依次完成去重、RRF 融合、Rerank 和元数据补全；
- 支持 PGVector 与 Milvus 两类向量存储实现。

![多路检索](assets/multi-channel-retrieval.png)

### MCP 工具调用

- 根据意图节点路由到对应 MCP 工具；
- 使用 LLM 提取参数并按照工具 Schema 校验；
- 缺少必填参数时触发澄清，参数非法或工具异常时返回降级结果；
- `mcp-server` 提供天气、票务、销售和联网搜索示例工具。

### 流量保护与可观测性

- 使用 Redis ZSET、分布式信号量、Pub/Sub 和 Lua 实现公平排队；
- 支持等待超时、客户端取消、异常释放和租约回收；
- 使用 SSE 分事件返回生成内容、引用来源和队列状态；
- 记录 RAG Run 与 Node 的输入、输出、耗时和异常。

## 项目结构

```text
ragent
├─ framework    通用 Web、异常、认证上下文、幂等、MQ、Trace 和 SSE
├─ infra-ai     Chat、Embedding、Rerank、VLM、模型路由和故障降级
├─ bootstrap    RAG 问答、知识库、入库 Pipeline、意图树和管理 API
├─ mcp-server   独立 MCP 工具服务
├─ frontend     React + Vite 管理端和聊天界面
└─ resources    数据库脚本、Docker Compose 和示例知识文档
```

![模块分层](assets/ragent-module-layering-v2.png)

## 技术栈

| 类型 | 技术 |
|---|---|
| 后端 | Java 17、Spring Boot、MyBatis-Plus、Sa-Token |
| AI 能力 | LLM、Embedding、Rerank、VLM、MCP Java SDK |
| 检索 | PGVector / Milvus、Elasticsearch、LightRAG |
| 中间件 | Redis、Redisson、RocketMQ、RustFS / S3 |
| 前端 | React、TypeScript、Vite、Tailwind CSS |
| 工程化 | Maven、Docker Compose、Trace、SSE |

项目没有直接使用 Spring AI 或 LangChain4j 的上层编排能力。模型、Embedding、Rerank、检索通道和问题重写均通过接口隔离，由业务 Pipeline 自行编排，以保留对流程、容错和并发策略的控制能力。

## 本地运行

### 环境要求

- JDK 17+
- Docker Desktop 与 Docker Compose
- Node.js 18+
- Maven Wrapper（仓库已包含）
- 至少一个可用的 Chat、Embedding 和 Rerank 模型服务

### 1. 克隆仓库

```bash
git clone https://github.com/JinChengZhan/ragent.git
cd ragent
```

### 2. 启动基础设施

```bash
docker compose -f resources/docker/postgresql-docker-compose.yaml up -d
docker compose -f resources/docker/redis-docker-compose.yaml up -d
docker compose -f resources/docker/rustfs-docker-compose.yaml up -d
```

RocketMQ 可根据本机架构选择 `resources/docker/` 下对应的 Compose 文件启动。

上述账号和密码仅用于本地开发，生产环境必须通过环境变量或密钥管理服务覆盖。

### 3. 初始化数据库

在数据库客户端中按顺序执行：

```text
resources/database/schema_pg.sql
resources/database/init_data_pg.sql
```

默认数据库信息：

```text
host: 127.0.0.1
port: 5432
database: ragent
username: postgres
password: postgres
```

### 4. 配置模型

模型候选、能力和路由配置位于：

```text
bootstrap/src/main/resources/application.yaml
```

根据选择的供应商配置对应环境变量，例如：

```powershell
$env:SILICONFLOW_API_KEY="your-api-key"
$env:BAILIAN_API_KEY="your-api-key"
$env:AIHUBMIX_API_KEY="your-api-key"
```

不要将真实 API Key 写入配置文件或提交到 Git。

### 5. 启动后端

Windows PowerShell：

```powershell
.\mvnw.cmd -DskipTests install
.\mvnw.cmd -pl bootstrap spring-boot:run
```

后端默认地址：`http://localhost:9090/api/ragent`

### 6. 启动前端

```bash
cd frontend
npm install
npm run dev
```

前端默认地址：`http://localhost:5173`

## 当前个人改动

- 固定 Maven Compiler、Dependency 和 Surefire 插件版本，降低本地构建差异；
- 将本地开发文档中的后端端口统一为 `9090`；
- 增加 PostgreSQL + PGVector、Redis 和 RustFS 的独立 Docker Compose 配置；
- 排除本地 PDF 渲染产物和误生成的根目录锁文件；
- 已在 Windows 环境完成五个 Maven 模块的编译验证。

## 后续计划

- [ ] 补充电商商品、订单和售后知识数据；
- [ ] 新增订单查询、物流查询和售后规则 MCP 工具；
- [ ] 构建电商问答测试集，记录 Recall@K、MRR 和响应耗时；
- [ ] 对不同检索组合进行可复现的对比实验；
- [ ] 完善自动化测试、启动脚本和部署说明。

## 验证

```powershell
.\mvnw.cmd -DskipTests compile
```

当前验证结果：`framework`、`infra-ai`、`bootstrap`、`mcp-server` 及父工程均编译成功。

Docker Compose 配置校验：

```bash
docker compose -f resources/docker/postgresql-docker-compose.yaml config --quiet
docker compose -f resources/docker/redis-docker-compose.yaml config --quiet
docker compose -f resources/docker/rustfs-docker-compose.yaml config --quiet
```

## 上游项目与许可证

- 上游仓库：[nageoffer/ragent](https://github.com/nageoffer/ragent)
- 上游文档：[Ragent AI 文档](https://nageoffer.com/ragent)
- 本项目保留原项目版权及署名，基于 [Apache License 2.0](LICENSE) 使用和分发。

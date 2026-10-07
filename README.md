# FlowServe — 电商智能客服 Agent

这是一个完整的 AI 客服系统，覆盖从对话交互到智能分类的端到端能力。用户可以通过自然语言查询订单、获取物流信息、获取政策解答，系统还能自动从历史对话中挖掘知识并做话题归类。

## 核心技术

- **Web 层**：FastAPI 提供 RESTful 接口和前端页面
- **AI 层**：LangGraph 编排 Agent 工作流，LangChain 集成 LLM 调用
- **数据层**：MySQL 存储业务数据，Milvus 负责向量检索
- **辅助存储**：MinIO 为 Milvus 提供对象存储支持
- **依赖管理**：uv 管理 Python 环境

系统使用三组独立上游接口（聊天、嵌入、重排），通过环境变量配置，切换供应商或模型无需修改代码。

## 启动流程

按顺序执行以下步骤：

```bash
# 安装 Python 依赖（需 3.12+）
uv sync

# 复制环境变量模板，填入 API 密钥
cp .env.example .env
# 编辑 .env，填写 CHAT_API_KEY、EMBED_API_KEY、RERANK_API_KEY

# 启动数据库和向量检索容器
docker compose up -d

# 导入测试数据
make seed
make seed-conv

# 构建知识库（切块 → 嵌入 → 写入 Milvus）
make kb-build
make kb-vectorize

# 启动应用服务
make dev
```

完成后访问 [http://localhost:8000](http://localhost:8000) 进入聊天界面。

完整部署文档请查看 [DEPLOY.md](./DEPLOY.md)。

## 服务端口

| 端口 | 用途 |
| --- | --- |
| 8000 | 主应用服务（FastAPI） |
| 8101 | 物流 MCP 服务 |
| 8102 | 售后 MCP 服务 |
| 8110 | 主题分类器推理服务 |
| 3000 | Langfuse 观测平台 |
| 19530 | Milvus 向量数据库 |
| 3306 | MySQL 数据库 |
| 9000/9001 | MinIO 对象存储 |

## 功能界面

系统提供以下管理页面：

- **聊天主界面** — 用户与客服 Agent 对话交互
- **知识库管理** — 文档上传、切块入库与检索验证
- **RAG 评估** — 四种检索策略对比报告
- **飞轮审核** — 自动挖掘数据的待审队列
- **数据观测** — 成本统计与置信度校准分析
- **主题分布** — 17 类问题量统计展示
- **分类器验收** — 模型训练与评测结果总览

## 项目结构

| 目录 | 说明 |
| --- | --- |
| `app/api/` | HTTP 路由与接口定义 |
| `app/graph/` | Agent 工作流编排（状态、节点、路由逻辑） |
| `app/core/` | 核心能力模块（检索、意图识别、摘要等） |
| `app/kb/` | 知识库流水线（文档切块、嵌入、双写） |
| `app/tools/` | 工具系统（内置工具定义、MCP 客户端、执行引擎） |
| `app/db/` | 数据库表模型与仓储层 |
| `app/static/` | 前端页面资源 |
| `mcp_servers/` | 业务 MCP 服务（物流、售后独立进程） |
| `sql/` | 数据库初始化与种子数据脚本 |
| `scripts/` | 离线处理脚本（评估、微调、飞轮批处理等） |

## 章节功能与验收

系统按课程章节逐步构建，每章有对应的验收命令：

| 章节 | 功能 | 测试命令 |
| --- | --- | --- |
| ch01 | 流式对话、结构化提取 | `make eval` |
| ch02 | 5 个业务工具、单轮 Function Calling | `make eval-agent` |
| ch03 | 知识库构建与对话挖掘 | `make kb-build` / `make kb-vectorize` |
| ch04 | 混合检索、重排与四策略评估 | `make smoke-rag` / `make eval-rag` |
| ch05 | LangGraph 工作流、Agent ReAct 循环 | `make eval-ch05` |
| ch06 | 意图分流、指代消解、退款子流程 | `make smoke-interrupt` / `make eval-ch06` |
| ch07 | 上下文管理（滑窗、摘要、前缀缓存） | `make eval-ch07` |
| ch08 | 工具系统、MCP 动态发现与审计日志 | `make eval-ch08` |
| ch09 | 可观测平台、数据飞轮、成本统计 | `make langfuse-up` / `make flywheel` |
| ch10 | 主题分类器训练与 ONNX 推理 | `make ch10-train` / `make ch10-eval` |

单测命令：`make test`（无需真实模型）

## 环境要求

- Python ≥ 3.12（`uv` 自动管理依赖，本机 Python 版本不影响）
- Docker + Docker Compose（MySQL、Milvus、etcd、MinIO 依赖容器运行）
- 内存 ≥ 8 GB，磁盘 ≥ 10 GB
- 有效的模型 API 密钥（硅基流动账号可跑通全课程，见 `.env.example`）

## 配置说明

在 `.env` 文件中配置三个 API 密钥（硅基流动账号共用）：

```bash
CHAT_API_KEY=sk-xxx        # 聊天模型
EMBED_API_KEY=sk-xxx       # 嵌入模型（BAAI/bge-m3）
RERANK_API_KEY=sk-xxx      # 重排模型（BAAI/bge-reranker-v2-m3）
```

默认 API 地址已内置，无需修改 `CHAT_BASE_URL`。思考链默认关闭（`CHAT_THINKING=disabled`），开启会显著增加响应耗时，影响流式体验和多步 Agent 演示。

## 常用操作

```bash
make help          # 查看所有可用命令及说明
make dev           # 启动完整服务
make dev-down      # 停止应用（依赖容器继续运行）
make mcp-up        # 单独启动两台 MCP 服务
make mcp-down      # 停止 MCP 服务
make kb-build      # 知识库切块落 MySQL
make kb-vectorize  # 知识库向量化写入 Milvus
make kb-reset      # 重置知识库（清 MySQL 两表 + 删除 Milvus collection）
make test          # 运行全部单测
docker compose up -d   # 仅启动依赖容器
```

## 验证方式

启动完成后，用以下命令验证系统是否正常工作：

```bash
printf '{"user_id":"u1","message":"订单 1001 的物流到哪了"}' > /tmp/q.json
curl -s --max-time 90 http://localhost:8000/api/agent \
     -H 'Content-Type: application/json' --data-binary @/tmp/q.json
```

预期返回中包含 `query_order` 和 `query_logistics` 两个工具调用标记，且 `answer` 字段出现物流状态信息。

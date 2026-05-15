# 工业 MiniApp —— 基于 xDM-F 的轻量级制造执行系统

> 一款面向 **PL120 精密行星减速机** 生产场景的轻量级 MES（制造执行系统），基于华为 iDME / xDM-F 数据引擎构建，集成 AI 智能助手，支持 RAG 知识检索与自然语言驱动的制造操作。

## 功能特性

- **设备管理** — 设备台账、状态管理、分类体系与扩展属性
- **物料/BOM 管理** — 物料信息维护、版本管理、BOM 结构、入库/出库/修订
- **物料分类** — 分类树管理、扩展属性定义
- **工艺路线** — 制造工艺路线定义与管理
- **工序管理** — 工序创建、设备绑定、工序执行跟踪
- **可视化工艺画布** — 无限画布 + 拖拽式工序节点编排，自动连线，多选绑定物料/生产设备/检验设备
- **生产计划** — 计划制定、工序执行进度跟踪
- **AI 智能制造助手** — 基于 DeepSeek v3.2 的 LLM Agent，支持：
  - 工具调用：自然语言驱动设备/物料/BOM 的增删改查与批量操作
  - RAG 知识检索：上传 PDF/Markdown 文档，PgVector 向量检索
  - SSE 流式响应，实时推送到前端
  - 对话记忆持久化（PostgreSQL）
  - 查询分类与改写

## 技术栈

### 后端

| 层级 | 技术 |
|------|------|
| 语言 | Java 17 |
| 框架 | Spring Boot 3.4.4 |
| 构建 | Maven |
| 数据引擎 | 华为 iDME / xDM-F（业务数据不直接访问数据库，全部通过 XdmfApiClient 调用 xDM-F Runtime API） |
| 数据库（AI 功能） | PostgreSQL + PgVector |
| ORM（对话记录） | MyBatis Plus 3.5.9 |
| AI 引擎 | DashScope（阿里云）+ Spring AI 1.0.0-M7 + Spring AI Alibaba 1.0.0-M6.1 |
| 大语言模型 | DeepSeek v3.2（对话）、text-embedding-v3（向量嵌入，1024 维） |
| Agent 框架 | LangChain4J DashScope 社区版 |
| API 文档 | SpringDoc OpenAPI + Knife4j |
| 其他 | Lombok、Fastjson、Hutool、Apache Tika、PDFBox、Guava、Jackson |

### 前端

| 层级 | 技术 |
|------|------|
| 框架 | Vue 3.4 + TypeScript 5.4 |
| 构建 | Vite 5.2 |
| 状态管理 | Pinia |
| 路由 | Vue Router 4 |
| 样式 | Tailwind CSS 3.4 |
| UI 组件 | Reka UI (Radix) + shadcn/ui 风格组件 + Lucide 图标 |
| AI 聊天 | Vercel AI SDK（@ai-sdk/vue）、vue-stream-markdown、Shiki 代码高亮、DOMPurify |
| 动画 | Motion-V |

## 项目结构

```
mini_industry_app_total/
├── mini_industry_backen/              # 后端（Spring Boot 3 / Java 17）
├── mini_app_front-branch-betterway/   # 前端（Vue 3 / TypeScript / Vite）
├── deploy/miniapp-web/                # 一键部署配置
├── docs/                              # 设计文档
├── pl120-equipment-maintenance.md     # PL120 设备维护手册
├── pl120-material-bom-knowledge.md    # PL120 物料/BOM 知识库
└── pl120-process-knowledge.md         # PL120 工艺知识库
```

### 后端模块结构

```
mini_industry_backen/src/main/java/com/huawei/idme/miniapp/
├── controller/          # 业务控制器（设备、物料、工序、工艺路线、生产计划等）
├── service/             # 业务逻辑层
├── assembler/           # DTO <-> xDM-F Map 转换器
├── dto/                 # 数据传输对象
├── util/                # 工具类（xDM-F 查询构建、JSON 处理、双键映射等）
├── config/              # 配置类（Demo 数据初始化、xDM-F 配置、CORS 等）
├── infra/xdmf/          # xDM-F 基础设施层
├── agent/               # AI Agent 层
│   ├── app/             # Agent 核心（ReAct 模式、系统提示词）
│   ├── config/          # ChatClient、RAG、向量存储配置
│   ├── controller/      # AI 对话控制器（SSE 流式）
│   ├── rag/             # RAG 管道（查询分类、改写、文档加载、检索增强）
│   ├── tools/           # Agent 工具（设备/物料的增删改查、批量操作）
│   └── chatmemory/      # PostgreSQL 对话记忆
└── scripts/             # 运维脚本（数据初始化、冒烟测试等）
```

### 前端模块结构

```
mini_app_front-branch-betterway/src/
├── views/               # 页面视图
│   ├── EquipmentView         # 设备管理
│   ├── MaterialView          # 物料/BOM 管理
│   ├── MaterialCategoryView  # 物料分类
│   ├── ProcessEntityView     # 可视化工艺画布
│   ├── ProcessRouteView      # 工艺路线
│   └── ProcedureView         # 工序管理
├── components/
│   ├── AIAssistant           # AI 智能助手
│   ├── KnowledgeBase         # 知识库管理
│   ├── ai-assistant/         # AI 聊天子组件
│   ├── ai-elements/          # AI 消息渲染组件
│   └── ui/                   # 通用 UI 组件
├── stores/              # Pinia 状态管理
├── router/              # 路由配置
└── types/               # TypeScript 类型定义
```

## 核心架构

本系统采用 **非传统 CRUD** 架构，业务数据不直接读写数据库：

```
Controller → Service → Assembler（DTO ↔ Map 转换）→ XdmfApiClient → 华为 iDME / xDM-F Runtime
```

PostgreSQL 仅用于 AI 聊天记录和 PgVector 向量存储，所有业务实体的持久化均通过 xDM-F 完成。

## 快速开始

### 环境要求

- JDK 17
- Maven 3.9+
- Node.js + npm
- PostgreSQL（仅 AI 功能需要）
- 华为 iDME / xDM-F 运行时环境

### 前端开发

```bash
cd mini_app_front-branch-betterway
npm install
npm run dev
# Vite 开发服务器启动于 port 3000，自动代理 /api /chat /ai 到 localhost:8080
```

### 后端开发

```bash
cd mini_industry_backen

# 设置环境变量
export SPRING_PROFILES_ACTIVE=dev
export XDMF_HOST=127.0.0.1:8003
export XDMF_APP_ID=your_app_id
export XDMF_AUTH_TOKEN=your_token
export DB_PASSWORD=your_pg_password

mvn spring-boot:run -Dspring-boot.run.profiles=dev
```

### 完整构建（前端嵌入后端）

```bash
# 1. 构建前端
cd mini_app_front-branch-betterway
npm ci && npm run build

# 2. 构建后端（Maven 自动将前端 dist/ 嵌入 classpath:/static/）
cd ../mini_industry_backen
mvn -q -DskipTests package

# 3. 运行
java -jar target/miniapp-1.0.0.jar
```

### 数据库初始化（仅 AI 功能）

方式一：激活 `ai-db-init` Profile：

```bash
export SPRING_PROFILES_ACTIVE=prod,ai-db-init
```

方式二：手动执行 SQL 脚本：

```bash
# 依次执行以下脚本
src/main/resources/db/postgresql/01-extensions.sql
src/main/resources/db/postgresql/02-chat_message.sql
src/main/resources/db/postgresql/03-vector_store.sql
```

### Spring Profile 说明

| Profile | 说明 |
|---------|------|
| `prod`（默认） | 生产模式，需要配置环境变量 |
| `dev` | 本地开发，使用占位符 |
| `no-ai` | 仅制造功能，排除 DashScope/PgVector/Agent |
| `ai-db-init` | 启动时自动执行 PostgreSQL DDL |

## 部署

### 一键部署

参考 [deploy/miniapp-web/](deploy/miniapp-web/) 目录：

```bash
# 1. 配置环境变量（复制 .env.example 为 .env 并填写）
cp deploy/miniapp-web/.env.example deploy/miniapp-web/.env

# 2. 构建并打包
cd mini_app_front-branch-betterway && npm ci && npm run build
cd ../mini_industry_backen && mvn -q -DskipTests package

# 3. 启动
java -jar mini_industry_backen/target/miniapp-1.0.0.jar
```

### 前端独立部署

前端支持独立部署到 Vercel，参见 `vercel.json` 和 `.github/workflows/deploy.yml`。

### 运维脚本

| 脚本 | 说明 |
|------|------|
| `scripts/create-exa-definitions.ps1` | 初始化扩展属性定义 |
| `scripts/create_classification_nodes_equipment.py` | 初始化设备分类节点 |
| `scripts/smoke.ps1` | 冒烟测试 |
| `scripts/test-equipment-create.ps1` | 设备创建测试 |
| `scripts/test-vector-store.ps1` | 向量存储测试 |

## AI 助手使用

系统内置 AI 智能制造助手，支持：

1. **自然语言查询** — "查询所有 CNC 加工中心"、"查看物料 PL120-A001 的 BOM 结构"
2. **自然语言操作** — "创建一台设备编号为 E007 的数控车床"、"将设备 E001 状态改为维护中"
3. **批量操作** — "批量更新设备 E001-E005 的状态为运行中"
4. **知识问答** — 上传工艺文档后，基于 RAG 进行知识检索和问答
5. **流式响应** — SSE 实时推送 AI 回复，支持 Markdown 渲染和代码高亮

## 文档

- [架构概述](mini_industry_backen/docs/#%20工业MiniApp：基于xDM-F的简易制造执行系统.md)
- [API 数据结构](mini_industry_backen/docs/API_Data_Structures.md)
- [Agent 工具开发指南](mini_industry_backen/docs/Agent_Tool_Development_Guide.md)
- [SSE 开发指南](mini_industry_backen/docs/Frontend_Agent_SSE_Development_Guide.md)
- [批量更新设计文档](mini_industry_backen/docs/Batch_Update_Design_Document.md)
- [向量存储测试指南](mini_industry_backen/docs/VectorStoreTestGuide.md)
- [数据库初始化指南](mini_industry_backen/docs/database-init.md)
- [xDM-F API 清单](mini_industry_backen/docs/xdm-f-runtime-api-inventory.md)

### PL120 知识库

- [设备维护手册](pl120-equipment-maintenance.md) — 6 台设备（E1-E6）的维护规范
- [物料/BOM 知识库](pl120-material-bom-knowledge.md) — 物料分类、BOM 结构、供应商管理
- [工艺知识库](pl120-process-knowledge.md) — 太阳轮制造工艺（5 道工序，37 分钟标准工时）

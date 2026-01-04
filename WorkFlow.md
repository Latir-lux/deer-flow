# DeerFlow 项目架构与工作流程分析

> **DeerFlow** (Deep Exploration and Efficient Research Flow) 是一个社区驱动的深度研究框架，基于 LangGraph 构建，通过多智能体协作完成深度研究、生成报告以及创建播客和演示文稿等内容。

---

## 目录

1. [项目层级结构](#1-项目层级结构)
2. [后端模块详解](#2-后端模块详解)
3. [前端模块详解](#3-前端模块详解)
4. [技术栈总览](#4-技术栈总览)
5. [环境配置指南](#5-环境配置指南)
6. [工作流程](#6-工作流程)
7. [多智能体协作模式](#7-多智能体协作模式)

---

## 1. 项目层级结构

DeerFlow 采用分层架构设计，主要分为以下四个层级：

```
┌─────────────────────────────────────────────────────────────────┐
│                       表示层 (Presentation Layer)                │
│                     Web UI (Next.js/React)                       │
├─────────────────────────────────────────────────────────────────┤
│                       接口层 (API Layer)                         │
│                     FastAPI Server + SSE                         │
├─────────────────────────────────────────────────────────────────┤
│                       核心业务层 (Core Business Layer)           │
│     Graph (LangGraph) + Agents + Prompts + Tools + RAG           │
├─────────────────────────────────────────────────────────────────┤
│                       基础设施层 (Infrastructure Layer)          │
│     LLM Providers + Crawler + TTS + MCP + Checkpointing          │
└─────────────────────────────────────────────────────────────────┘
```

### 1.1 层级概览

| 层级 | 描述 | 主要目录 |
|------|------|----------|
| **表示层** | 用户界面，提供交互式Web体验 | `web/` |
| **接口层** | RESTful API 和 SSE 流式接口 | `src/server/` |
| **核心业务层** | 多智能体系统、工作流编排、提示工程 | `src/agents/`, `src/graph/`, `src/prompts/`, `src/tools/` |
| **基础设施层** | LLM集成、网页爬取、语音合成、持久化 | `src/llms/`, `src/crawler/`, `src/rag/` |

---

## 2. 后端模块详解

### 2.1 入口模块

#### `main.py` - 控制台入口
- **作用**: 提供命令行交互界面，支持内置问题选择和自定义查询
- **技术栈**: Python, InquirerPy, asyncio
- **功能**:
  - 多语言支持 (英文/中文)
  - 交互式问题选择
  - 调试模式开关

#### `server.py` - Web服务器入口
- **作用**: 启动 FastAPI 服务器
- **技术栈**: Python, uvicorn, FastAPI
- **功能**:
  - 热重载支持
  - 跨平台事件循环兼容
  - 优雅关闭处理

#### `workflow.py` - 工作流编排
- **作用**: 协调整个研究工作流程的执行
- **技术栈**: Python, LangGraph
- **功能**:
  - 异步工作流执行
  - 状态管理
  - MCP 工具配置

---

### 2.2 智能体模块 (`src/agents/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `agents.py` | 智能体工厂，使用 ReAct 模式创建智能体 | LangGraph, LangChain |
| `tool_interceptor.py` | 工具调用拦截器，支持工具执行前中断 | Python |

**核心智能体角色**:

| 智能体 | 职责 | 工具权限 |
|--------|------|----------|
| **Coordinator** | 入口协调器，处理问候、分流任务 | handoff_to_planner, direct_response |
| **Planner** | 研究规划器，分解任务为结构化步骤 | 无工具（纯LLM推理） |
| **Researcher** | 信息收集者，执行网络搜索和爬取 | web_search, crawl_tool, local_search |
| **Analyst** | 数据分析师，综合分析收集的信息 | 无工具（纯LLM推理） |
| **Coder** | 代码执行者，运行Python代码进行计算 | python_repl_tool |
| **Reporter** | 报告撰写者，生成最终研究报告 | 无工具（纯LLM推理） |

---

### 2.3 图工作流模块 (`src/graph/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `builder.py` | 构建 StateGraph 工作流图 | LangGraph StateGraph |
| `nodes.py` | 定义各个节点的处理逻辑 | LangChain, LangGraph |
| `types.py` | 定义状态数据结构 | Pydantic, dataclasses |
| `checkpoint.py` | 检查点管理，支持会话持久化 | MongoDB, PostgreSQL |
| `utils.py` | 图工作流工具函数 | Python |

**工作流节点**:
```
START → coordinator → background_investigator → planner → research_team → reporter → END
                                                    ↑              ↓
                                                    └── researcher/analyst/coder ←─┘
```

---

### 2.4 工具模块 (`src/tools/`)

| 文件/目录 | 作用 | 技术栈 |
|-----------|------|--------|
| `search.py` | 多引擎网络搜索工具 | Tavily, Brave, DuckDuckGo, Serper, Arxiv, Wikipedia |
| `crawl.py` | 网页内容爬取工具 | Jina, Readability |
| `python_repl.py` | Python REPL 代码执行 | LangChain Experimental |
| `tts.py` | 文本转语音服务 | Volcengine TTS API |
| `retriever.py` | RAG 检索工具 | LangChain |
| `tavily_search/` | Tavily 搜索增强实现 | Tavily API |
| `infoquest_search/` | InfoQuest 搜索集成 | BytePlus InfoQuest |

**支持的搜索引擎**:
- Tavily (推荐，支持图片)
- InfoQuest (字节跳动)
- Brave Search
- DuckDuckGo
- Google Serper
- Searx
- Arxiv
- Wikipedia

---

### 2.5 LLM 模块 (`src/llms/`)

| 文件/目录 | 作用 | 技术栈 |
|-----------|------|--------|
| `llm.py` | LLM 实例管理和配置加载 | LangChain, LiteLLM |
| `providers/dashscope.py` | 阿里云通义千问适配器 | Dashscope API |

**支持的 LLM 类型**:
- `basic`: 基础任务处理
- `reasoning`: 推理任务（规划、分析）
- `vision`: 视觉理解
- `code`: 代码生成

**支持的模型提供商**:
- OpenAI (GPT-4o)
- DeepSeek (DeepSeek-V3)
- Google (Gemini 2.0/2.5)
- 阿里云 (Qwen-Max, Qwen3)
- 字节跳动 (Doubao)
- Azure OpenAI
- Ollama (本地模型)

---

### 2.6 爬虫模块 (`src/crawler/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `crawler.py` | 统一爬虫接口，支持多引擎切换 | Python |
| `jina_client.py` | Jina Reader API 客户端 | Jina AI |
| `readability_extractor.py` | 本地可读性提取器 | ReadabiliPy |
| `infoquest_client.py` | InfoQuest 爬取客户端 | BytePlus |
| `article.py` | 文章数据模型 | Pydantic |

---

### 2.7 RAG 模块 (`src/rag/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `retriever.py` | RAG 检索器抽象基类 | LangChain |
| `builder.py` | RAG 提供商构建器 | Python |
| `ragflow.py` | RAGFlow 集成 | RAGFlow API |
| `milvus.py` | Milvus 向量数据库集成 | pymilvus, LangChain-Milvus |
| `qdrant.py` | Qdrant 向量数据库集成 | qdrant-client |
| `vikingdb_knowledge_base.py` | 火山引擎 VikingDB 集成 | VikingDB API |
| `dify.py` | Dify 平台集成 | Dify API |
| `moi.py` | MOI 集成 | MOI API |

---

### 2.8 提示词模块 (`src/prompts/`)

| 文件 | 作用 |
|------|------|
| `coordinator.md` | 协调器智能体提示词模板 |
| `planner.md` | 规划器智能体提示词模板 |
| `researcher.md` | 研究者智能体提示词模板 |
| `analyst.md` | 分析师智能体提示词模板 |
| `coder.md` | 编码器智能体提示词模板 |
| `reporter.md` | 报告者智能体提示词模板 |
| `planner_model.py` | 计划数据模型 (Plan, Step) |
| `template.py` | Jinja2 模板渲染引擎 |
| `*.zh_CN.md` | 中文版本提示词 |

**提示词模板支持**:
- 多语言 (en-US, zh-CN)
- Jinja2 模板语法
- 动态变量注入 (时间、配置等)
- 多种报告风格 (学术、新闻、科普、社交媒体、投资分析)

---

### 2.9 配置模块 (`src/config/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `configuration.py` | 运行时配置管理 | dataclasses |
| `loader.py` | YAML/环境变量加载器 | PyYAML, python-dotenv |
| `agents.py` | 智能体-LLM 映射配置 | Python |
| `tools.py` | 工具配置 (搜索引擎、爬虫引擎等) | Python |
| `report_style.py` | 报告风格枚举 | Python Enum |
| `questions.py` | 内置示例问题 | Python |

---

### 2.10 服务器模块 (`src/server/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `app.py` | FastAPI 应用主文件 | FastAPI, SSE-Starlette |
| `chat_request.py` | 聊天请求数据模型 | Pydantic |
| `mcp_request.py` | MCP 请求处理 | Python |
| `mcp_utils.py` | MCP 工具加载器 | langchain-mcp-adapters |
| `rag_request.py` | RAG 请求处理 | Pydantic |
| `eval_request.py` | 报告评估请求 | Pydantic |
| `config_request.py` | 配置API响应 | Pydantic |

**API 端点**:
- `POST /api/chat` - 流式研究对话
- `POST /api/podcast` - 播客生成
- `POST /api/ppt` - PPT 生成
- `POST /api/prose` - 文章改写
- `POST /api/tts` - 文本转语音
- `GET /api/config` - 获取配置
- `POST /api/evaluate` - 报告评估

---

### 2.11 评估模块 (`src/eval/`)

| 文件 | 作用 | 技术栈 |
|------|------|--------|
| `evaluator.py` | 报告综合评估器 | Python |
| `llm_judge.py` | LLM 作为评判者 | LangChain |
| `metrics.py` | 自动化评估指标 | Python |

**评估维度**:
- 内容完整性
- 信息准确性
- 结构清晰度
- 语言质量
- 引用规范

---

### 2.12 内容生成模块

#### 播客生成 (`src/podcast/`)
| 文件 | 作用 |
|------|------|
| `graph/builder.py` | 播客工作流图构建 |
| `graph/script_writer_node.py` | 脚本编写节点 |
| `graph/tts_node.py` | 语音合成节点 |
| `graph/audio_mixer_node.py` | 音频混合节点 |

#### PPT 生成 (`src/ppt/`)
| 文件 | 作用 |
|------|------|
| `graph/builder.py` | PPT 工作流图构建 |
| `graph/ppt_composer_node.py` | Marp 脚本生成 |
| `graph/ppt_generator_node.py` | PDF/PPT 转换 |

#### 文章改写 (`src/prose/`)
| 文件 | 作用 |
|------|------|
| `graph/builder.py` | 文章处理工作流 |
| `graph/prose_*_node.py` | 续写/改进/缩短/扩展/修复/精简 |

#### 提示词增强 (`src/prompt_enhancer/`)
| 文件 | 作用 |
|------|------|
| `graph/builder.py` | 提示词增强工作流 |
| `graph/enhancer_node.py` | 提示词优化节点 |

---

## 3. 前端模块详解

### 3.1 目录结构

```
web/
├── src/
│   ├── app/              # Next.js 页面和 API 路由
│   ├── components/       # UI 组件库
│   │   ├── deer-flow/    # 业务组件
│   │   ├── editor/       # 富文本编辑器
│   │   ├── magicui/      # 动画组件
│   │   └── ui/           # 基础 UI 组件 (shadcn/ui)
│   ├── core/             # 核心工具库
│   │   ├── api/          # API 客户端
│   │   ├── store/        # 状态管理 (Zustand)
│   │   ├── sse/          # SSE 流处理
│   │   ├── mcp/          # MCP 客户端
│   │   ├── markdown/     # Markdown 处理
│   │   └── replay/       # 回放功能
│   ├── hooks/            # React Hooks
│   ├── lib/              # 工具函数
│   ├── styles/           # 全局样式
│   └── typings/          # TypeScript 类型定义
├── messages/             # 国际化消息
└── public/               # 静态资源
```

### 3.2 核心技术栈

| 类别 | 技术 |
|------|------|
| **框架** | Next.js 15 (App Router, Turbopack) |
| **语言** | TypeScript, React 19 |
| **状态管理** | Zustand + Immer |
| **UI 组件** | Radix UI + shadcn/ui |
| **样式** | Tailwind CSS + Tailwind Typography |
| **表单** | React Hook Form + Zod |
| **Markdown** | react-markdown + remark-gfm + rehype-katex |
| **富文本** | TipTap (基于 ProseMirror) |
| **动画** | Framer Motion |
| **图表** | @xyflow/react (React Flow) |
| **国际化** | next-intl |
| **主题** | next-themes |

---

## 4. 技术栈总览

### 4.1 后端技术栈

```yaml
核心框架:
  - Python: 3.12+
  - LangGraph: 0.3.5+ (工作流编排)
  - LangChain: 1.0+ (LLM 抽象层)
  - FastAPI: 0.110+ (Web 框架)

LLM 集成:
  - langchain-openai: OpenAI/Azure 模型
  - langchain-deepseek: DeepSeek 模型
  - langchain-google-genai: Google Gemini
  - litellm: 统一 LLM 接口

数据存储:
  - MongoDB: 会话检查点 (langgraph-checkpoint-mongodb)
  - PostgreSQL: 会话检查点 (langgraph-checkpoint-postgres)
  - Milvus: 向量数据库 (pymilvus)
  - Qdrant: 向量数据库 (qdrant-client)

工具集成:
  - httpx: HTTP 客户端
  - duckduckgo-search: 搜索引擎
  - arxiv: 学术论文搜索
  - wikipedia: 维基百科查询
  - mcp: Model Context Protocol

内容处理:
  - markdownify: HTML 转 Markdown
  - readabilipy: 可读性提取
  - jinja2: 模板引擎
  - json-repair: JSON 修复

数据处理:
  - pandas: 数据分析
  - numpy: 数值计算
  - yfinance: 金融数据
```

### 4.2 前端技术栈

```yaml
核心框架:
  - Next.js: 15.4+ (React 框架)
  - React: 19.0+ (UI 库)
  - TypeScript: 类型安全

UI 组件:
  - Radix UI: 无障碍原语组件
  - shadcn/ui: 组件库
  - Lucide React: 图标库
  - Framer Motion: 动画库

内容渲染:
  - react-markdown: Markdown 渲染
  - remark-gfm: GFM 支持
  - remark-math: 数学公式
  - rehype-katex: KaTeX 渲染
  - highlight.js: 代码高亮

编辑器:
  - TipTap: 富文本编辑器
  - Novel: 所见即所得编辑器

状态管理:
  - Zustand: 全局状态
  - Immer: 不可变更新
  - React Hook Form: 表单状态

工具库:
  - zod: 模式验证
  - nanoid: ID 生成
  - dompurify: XSS 防护
  - file-saver: 文件下载
  - docx: Word 文档生成
  - jspdf: PDF 生成
```

---

## 5. 环境配置指南

### 5.1 系统要求

| 组件 | 最低版本 | 推荐版本 |
|------|----------|----------|
| Python | 3.12 | 3.12+ |
| Node.js | 22 | 22+ |
| pnpm | 8 | 9+ |
| uv | 0.6.15 | 最新版 |

### 5.2 后端环境配置

#### 步骤 1: 安装依赖
```bash
# 克隆仓库
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow

# 使用 uv 安装依赖 (自动创建虚拟环境)
uv sync

# 安装开发依赖
uv pip install -e ".[dev]"

# 安装测试依赖
uv pip install -e ".[test]"
```

#### 步骤 2: 配置环境变量 (`.env`)
```bash
cp .env.example .env
```

**关键环境变量**:
```env
# 搜索引擎 API (选择一个)
TAVILY_API_KEY=your_tavily_api_key
BRAVE_SEARCH_API_KEY=your_brave_api_key
SEARCH_ENGINE=tavily  # tavily, brave_search, duckduckgo, serper, infoquest

# 爬虫引擎
CRAWLER_ENGINE=jina  # jina, readability, infoquest

# 检查点存储 (可选)
LANGGRAPH_CHECKPOINT_SAVER=true
LANGGRAPH_CHECKPOINT_DB_URL=postgresql://user:pass@localhost:5432/deerflow

# 语音合成 (可选)
TTS_PROVIDER=volcengine
VOLCENGINE_TTS_APPID=your_app_id
VOLCENGINE_TTS_ACCESS_TOKEN=your_token

# Python REPL (安全考虑，默认禁用)
ENABLE_PYTHON_REPL=false

# 调试模式
DEBUG=false

# LangSmith 追踪 (可选)
LANGSMITH_API_KEY=your_langsmith_key
LANGCHAIN_TRACING_V2=true
```

#### 步骤 3: 配置模型 (`conf.yaml`)
```bash
cp conf.yaml.example conf.yaml
```

**配置示例**:
```yaml
# 基础模型配置
BASIC_MODEL:
  base_url: "https://api.openai.com/v1"
  model: "gpt-4o"
  api_key: YOUR_API_KEY
  temperature: 0.7

# 推理模型配置 (用于规划器)
REASONING_MODEL:
  base_url: "https://api.deepseek.com"
  model: "deepseek-chat"
  api_key: YOUR_API_KEY

# 代码模型配置 (用于编码器)
CODE_MODEL:
  base_url: "https://api.openai.com/v1"
  model: "gpt-4o"
  api_key: YOUR_API_KEY

# 搜索引擎配置
SEARCH_ENGINE:
  include_domains: []
  exclude_domains: []
  search_depth: "advanced"
  include_images: true
```

### 5.3 前端环境配置

```bash
cd web

# 安装依赖
pnpm install

# 开发模式运行
pnpm dev
```

### 5.4 快速启动

```bash
# macOS/Linux
./bootstrap.sh -d

# Windows
bootstrap.bat -d
```

### 5.5 Docker 部署

```bash
# 构建镜像
docker build -t deer-flow-api .

# 使用 Docker Compose
docker compose up -d
```

---

## 6. 工作流程

### 6.1 完整研究工作流

```mermaid
graph TD
    A[用户输入] --> B[Coordinator 协调器]
    B -->|问候/闲聊| C[直接响应]
    B -->|研究任务| D[Background Investigator 背景调查]
    D --> E[Planner 规划器]
    E --> F{计划审核}
    F -->|需要修改| E
    F -->|批准| G[Research Team 研究团队]
    G --> H{步骤类型}
    H -->|research| I[Researcher 研究者]
    H -->|analysis| J[Analyst 分析师]
    H -->|processing| K[Coder 编码器]
    I --> L{更多步骤?}
    J --> L
    K --> L
    L -->|是| H
    L -->|否| M[Reporter 报告者]
    M --> N[最终报告]
    N -->|可选| O[Podcast 播客生成]
    N -->|可选| P[PPT 演示文稿]
```

### 6.2 各节点详细流程

#### 1. Coordinator (协调器)
```
输入: 用户消息
↓
分类请求类型:
  - 问候/闲聊 → direct_response() → 结束
  - 不当请求 → 礼貌拒绝 → 结束
  - 模糊问题 → 澄清对话 (最多3轮)
  - 研究问题 → handoff_to_planner()
↓
输出: 移交至规划器或直接响应
```

#### 2. Background Investigator (背景调查)
```
输入: 研究主题
↓
执行初步网络搜索
收集背景上下文信息
↓
输出: 背景调查结果
```

#### 3. Planner (规划器)
```
输入: 研究主题 + 背景信息
↓
评估现有上下文是否充足
↓
生成结构化研究计划:
  - 步骤列表 (research/analysis/processing)
  - 每步目标和预期输出
  - 搜索需求标记
↓
输出: Plan 对象
```

#### 4. Research Team (研究团队)
根据步骤类型分发任务:

**Researcher (研究者)**:
```
输入: 研究步骤
↓
执行工具调用:
  - web_search: 网络搜索
  - crawl_tool: 网页爬取
  - local_search: 本地知识库检索
↓
输出: Markdown 格式研究发现
```

**Analyst (分析师)**:
```
输入: 分析步骤 + 已收集信息
↓
纯 LLM 推理:
  - 交叉验证
  - 综合分析
  - 模式识别
↓
输出: 分析结论
```

**Coder (编码器)**:
```
输入: 处理步骤
↓
执行 Python 代码:
  - 数据计算
  - 数据可视化
  - 文件处理
↓
输出: 计算结果
```

#### 5. Reporter (报告者)
```
输入: 所有研究发现 + 分析结果
↓
根据报告风格生成:
  - academic: 学术论文风格
  - news: 新闻报道风格
  - popular_science: 科普文章风格
  - social_media: 社交媒体风格
  - strategic_investment: 投资分析风格
↓
输出: 完整 Markdown 报告
```

### 6.3 状态管理

```python
class State(MessagesState):
    # 运行时变量
    locale: str                        # 语言设置
    research_topic: str                # 原始研究主题
    clarified_research_topic: str      # 澄清后的主题
    observations: list[str]            # 观察记录
    resources: list[Resource]          # 资源列表
    current_plan: Plan                 # 当前计划
    final_report: str                  # 最终报告
    
    # 澄清状态
    enable_clarification: bool         # 是否启用澄清
    clarification_rounds: int          # 澄清轮次
    clarification_history: list[str]   # 澄清历史
    
    # 背景调查
    enable_background_investigation: bool
    background_investigation_results: str
    
    # 工作流控制
    goto: str                          # 下一节点
```

---

## 7. 多智能体协作模式

### 7.1 ReAct 模式

DeerFlow 使用 ReAct (Reasoning + Acting) 模式，智能体通过以下循环工作:

```
思考 (Reasoning) → 行动 (Acting) → 观察 (Observation) → 思考 ...
```

### 7.2 工具调用流程

```
1. LLM 决定调用工具
2. 工具拦截器检查 (interrupt_before_tools)
3. 执行工具调用
4. 返回结果给 LLM
5. LLM 处理结果，决定下一步
```

### 7.3 人机协作 (Human-in-the-Loop)

```
Planner 生成计划
    ↓
Human Feedback 节点
    ↓
用户选择:
  - 批准 → 继续执行
  - 修改 → 返回规划器
  - 拒绝 → 终止工作流
```

### 7.4 MCP (Model Context Protocol) 集成

```yaml
mcp_settings:
  servers:
    mcp-github-trending:
      transport: stdio
      command: uvx
      args: ["mcp-github-trending"]
      enabled_tools: ["get_github_trending_repositories"]
      add_to_agents: ["researcher"]
```

---

## 附录: 快速参考

### 常用命令

```bash
# 后端
uv run main.py              # 控制台 UI
uv run server.py --reload   # 开发服务器
make test                   # 运行测试
make lint                   # 代码检查
make format                 # 代码格式化
make langgraph-dev          # LangGraph Studio

# 前端
pnpm dev                    # 开发服务器
pnpm build                  # 生产构建
pnpm lint                   # ESLint 检查
pnpm typecheck              # TypeScript 检查
```

### 文件路径速查

| 功能 | 路径 |
|------|------|
| 主入口 | `main.py`, `server.py` |
| 智能体定义 | `src/agents/agents.py` |
| 工作流图 | `src/graph/builder.py` |
| 节点逻辑 | `src/graph/nodes.py` |
| 提示词模板 | `src/prompts/*.md` |
| 工具实现 | `src/tools/` |
| API 端点 | `src/server/app.py` |
| 前端页面 | `web/src/app/` |
| UI 组件 | `web/src/components/` |

---

*文档生成时间: 2026年1月4日*
*DeerFlow 版本: 0.1.0*

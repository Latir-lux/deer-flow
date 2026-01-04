# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**DeerFlow** (Deep Exploration and Efficient Research Flow) is a community-driven deep research framework built on LangGraph and LangChain. It orchestrates specialized AI agents to conduct comprehensive research, analyze data, execute code, and generate polished reports with support for podcast and PowerPoint generation.

**Tech Stack:**
- Backend: Python 3.12+, FastAPI, LangGraph, LangChain
- Frontend: Next.js 14+, TypeScript, React, TailwindCSS
- Package Management: `uv` (Python), `pnpm` (Node.js)
- Testing: pytest (Python), Jest (JavaScript)

## Common Commands

### Development Setup
```bash
# Install Python dependencies (uv handles venv creation automatically)
uv sync

# Install development and test dependencies
uv pip install -e ".[dev]" && uv pip install -e ".[test]"

# Configure environment
cp .env.example .env
cp conf.yaml.example conf.yaml

# Install frontend dependencies
cd web && pnpm install
```

### Running the Application
```bash
# Console UI (interactive or direct query)
uv run main.py
uv run main.py --interactive
uv run main.py "Your research question"

# Backend API server (development mode with reload)
uv run server.py --reload
# Or using Make
make serve

# Full stack (backend + frontend)
./bootstrap.sh -d    # macOS/Linux
bootstrap.bat -d     # Windows

# Frontend only (after backend is running)
cd web && pnpm dev
```

### Testing
```bash
# Run all Python tests
make test
# Or directly
uv run pytest tests/

# Run specific test file
pytest tests/integration/test_workflow.py

# Run with coverage report
make coverage

# Frontend tests
cd web && pnpm test:run

# Frontend linting and type checking
cd web && pnpm lint
cd web && pnpm typecheck
```

### Code Quality
```bash
# Format Python code with Ruff
make format

# Lint and auto-fix Python code
make lint

# Frontend linting, type checking, tests, and build
make lint-frontend
```

### Debugging
```bash
# LangGraph Studio for visual workflow debugging
make langgraph-dev
# Then open: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

# Enable debug logging
uv run main.py --debug "Your query"

# Or set environment variable
DEBUG=true
```

## Architecture Overview

### Multi-Agent Workflow System

DeerFlow uses **LangGraph StateGraph** to orchestrate a sophisticated multi-agent research pipeline:

```
User Query
    ↓
Coordinator (classifies intent, determines workflow)
    ↓
Background Investigator (optional: gathers context via web search)
    ↓
Planner (creates structured research plan with steps)
    ↓
[Human-in-the-loop: Plan review and modification]
    ↓
Research Team (parallel execution of plan steps)
    ├─ Researcher (web search, crawling, RAG retrieval)
    ├─ Analyst (data synthesis and analysis)
    └─ Coder (Python REPL for data processing)
    ↓
Reporter (aggregates findings, generates styled report)
    ↓
[Optional] Content Generation
    ├─ Podcast (script → TTS → audio)
    ├─ PowerPoint (slides + layout)
    └─ Prose Enhancement
```

**Key architectural patterns:**
- **State-based workflow**: LangGraph manages immutable state transitions across nodes
- **Agent factory pattern**: `create_react_agent()` with tool interception and per-agent LLM type mapping
- **Tool abstraction**: Unified interface for search, crawl, RAG, Python REPL, TTS
- **Provider abstraction**: Pluggable backends for LLMs, search engines, vector DBs, crawlers
- **Checkpoint persistence**: Optional PostgreSQL/MongoDB for conversation history

### Agent-LLM Mapping

Each agent type can be assigned a different LLM model tier (configured in `src/config/agents.py`):
- **reasoning**: Complex planning and coordination (e.g., Coordinator, Planner)
- **basic**: Standard tasks (e.g., Reporter)
- **code**: Code execution and analysis (e.g., Coder)
- **vision**: Image understanding (future use)

Configure models in `conf.yaml` with separate sections: `BASIC_MODEL`, `REASONING_MODEL`, `CODE_MODEL`, `VISION_MODEL`.

### State Management

**Core state structure** (`src/graph/types.py`):
- Message history (conversation turns)
- Tool call results
- Research plan (steps with status tracking)
- Clarification state (multi-turn refinement)
- Iteration tracking for planning loops

State flows through nodes and is preserved via checkpoints (default: in-memory, optional: PostgreSQL/MongoDB).

### Tool System

**Available tools** (configured per agent):
1. **Web Search**: Tavily (default), InfoQuest (recommended), Brave, DuckDuckGo, Arxiv, Searx, Serper
2. **Web Crawl**: Jina (default), InfoQuest, or readability-based extraction
3. **Python REPL**: Code execution in sandboxed environment (configurable via `ENABLE_PYTHON_REPL`)
4. **RAG Retriever**: Milvus, Qdrant, RAGFlow, Dify, VikingDB, MOI
5. **TTS**: Volcengine text-to-speech
6. **MCP Tools**: Dynamically loaded from MCP servers (configurable via `ENABLE_MCP_SERVER_CONFIGURATION`)

**Tool interception**: Pre-execution approval mechanism via `wrap_tools_with_interceptor()` for user control over sensitive operations.

## Directory Structure Highlights

### Backend (`src/`)
- **`agents/`**: Agent factory, tool interception
- **`config/`**: YAML config loading, agent-LLM mapping, tool config
- **`graph/`**: LangGraph workflow (builder, nodes, types, utils) - **~1,918 LOC**
  - `nodes.py` (~1,277 LOC): Core workflow logic
  - `builder.py`: Graph construction and wiring
  - `types.py`: State type definitions
- **`llms/`**: LLM factory with caching, provider integrations (OpenAI, DeepSeek, Google, Dashscope, Azure)
- **`tools/`**: Search, crawl, Python REPL, retriever, TTS implementations
- **`rag/`**: RAG provider factory and implementations (Milvus, Qdrant, RAGFlow, etc.)
- **`prompts/`**: Jinja2 templates with multi-locale support (`*.en_US.md`, `*.zh_CN.md`)
- **`server/`**: FastAPI app, request models, SSE streaming, MCP utils
- **`podcast/`, `ppt/`, `prose/`, `prompt_enhancer/`**: Specialized content generation workflows
- **`crawler/`**: Jina/InfoQuest client, readability extraction
- **`eval/`**: Evaluation framework (LLM judge, metrics)
- **`utils/`**: JSON repair, context management, log sanitization

### Frontend (`web/src/`)
- **`app/`**: Next.js app router pages
- **`components/`**: React components (Radix UI, Ant Design icons)
- **`core/`**: API client, utilities
- **`hooks/`**: Custom React hooks
- **`typings/`**: TypeScript type definitions

### Configuration Files
- **`conf.yaml`**: LLM model configurations, search/crawl engine settings
- **`.env`**: API keys, search engines, RAG providers, checkpointing, MCP servers
- **`langgraph.json`**: LangGraph CLI configuration for LangGraph Studio
- **`pyproject.toml`**: Python dependencies, pytest config, Ruff config

## Configuration System

### LLM Configuration (`conf.yaml`)
Supports OpenAI-compatible APIs via litellm format:
```yaml
BASIC_MODEL:
  base_url: https://api.provider.com/v1
  model: model-name
  api_key: YOUR_API_KEY
  # Optional:
  temperature: 0.6
  top_p: 0.90
  token_limit: 128000  # For context management
  verify_ssl: false    # For self-signed certs (dev only)

# Optional specialized models:
REASONING_MODEL: {...}
CODE_MODEL: {...}
VISION_MODEL: {...}
```

**Supported platforms:**
- OpenAI, Azure OpenAI, DeepSeek, Google Gemini (via `platform: "google_aistudio"`), Qwen (Aliyun DashScope), Volcengine Doubao
- Ollama (local: `http://localhost:11434/v1`)
- OpenRouter (prefix model with `openrouter/`)

**Note:** Non-reasoning models only (o1/o3/R1 not supported yet). Gemma-3 unsupported due to lack of tool usage.

### Environment Variables (`.env`)
Key variables:
- **Search**: `SEARCH_API` (tavily/infoquest/brave_search/duckduckgo/arxiv/serper), `TAVILY_API_KEY`, `INFOQUEST_API_KEY`
- **RAG**: `RAG_PROVIDER` (milvus/qdrant/ragflow/dify/moi), provider-specific configs
- **Checkpointing**: `LANGGRAPH_CHECKPOINT_SAVER=true`, `LANGGRAPH_CHECKPOINT_DB_URL`
- **Security**: `ENABLE_MCP_SERVER_CONFIGURATION`, `ENABLE_PYTHON_REPL`
- **Tracing**: `LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY`
- **Debug**: `DEBUG=true`

### Search Engine Configuration (`conf.yaml`)
```yaml
SEARCH_ENGINE:
  engine: tavily  # or infoquest (recommended)
  include_domains: [trusted-news.com]  # Whitelist
  exclude_domains: [spam.com]          # Blacklist
  include_images: true
  min_score_threshold: 0.4             # Filter low-quality results
  max_content_length_per_page: 5000    # Truncate content
```

### Web Search Toggle
Disable web search for offline/RAG-only mode:
```yaml
ENABLE_WEB_SEARCH: false
```

## Important Development Notes

### LangGraph Workflow Development
- Each workflow is a `StateGraph` with typed state
- Nodes are async functions that receive state and return partial updates
- Edges define routing logic (conditional or direct)
- Use `langgraph dev` to debug workflows visually in LangGraph Studio
- Checkpoints enable state recovery and conversation replay

### Adding New Components

**New Agent:**
1. Define in `src/agents/agents.py` or dedicated file
2. Add to `AGENT_LLM_MAP` in `src/config/agents.py`
3. Create prompt template in `src/prompts/`
4. Wire into graph in `src/graph/builder.py`

**New Tool:**
1. Implement with `@tool` decorator in `src/tools/`
2. Register in agent's tool list
3. Optionally add to tool interception list

**New RAG Provider:**
1. Create provider class in `src/rag/`
2. Implement `Retriever` interface
3. Register in `src/rag/builder.py`

**New LLM Provider:**
1. Add provider logic in `src/llms/providers/`
2. Update `_create_llm_use_conf()` in `src/llms/llm.py`

### Prompt Engineering
- Templates in `src/prompts/` use Jinja2 syntax
- Multi-locale support: `agent_name.{en_US,zh_CN}.md`
- Fallback to English if locale unavailable
- Dynamic variables: current time, research topic, plan details, agent context

### Human-in-the-Loop
- Plan review: System presents plan before execution
- Feedback: Accept with `[ACCEPTED]` or edit with `[EDIT PLAN] ...`
- Auto-acceptance: Set `auto_accepted_plan: true` in API request or CLI flag
- Web UI: Interactive plan modification before research begins

### Multi-Turn Clarification
- Disabled by default
- Enable via CLI: `--enable-clarification --max-clarification-rounds 3`
- Enable via API: `{"enable_clarification": true, "max_clarification_rounds": 3}`
- Coordinator asks clarifying questions for vague topics before starting research

### Checkpointing and Persistence
- Default: `MemorySaver` (in-memory, no persistence)
- PostgreSQL: Requires `langgraph-checkpoint-postgres==2.0.21` (note: avoid 2.0.23 due to serialization bug)
- MongoDB: Use `langgraph-checkpoint-mongodb`
- Set `LANGGRAPH_CHECKPOINT_DB_URL` in `.env`
- Collections auto-created: `checkpoint_writes_aio`, `checkpoints_aio`, `chat_streams`

### Security Considerations
- Never commit API keys (use `.env` files)
- `ENABLE_PYTHON_REPL=false` to disable code execution
- `ENABLE_MCP_SERVER_CONFIGURATION=false` to restrict MCP server loading
- Tool interception for user approval before sensitive operations
- Validate and sanitize user inputs (see `src/utils/json_utils.py`, `log_sanitizer.py`)

### Content Generation Pipelines
- **Podcast**: Report → Script → TTS (Volcengine) → Audio
- **PPT**: Report → Slides + Layout (Marp CLI required: `brew install marp-cli`)
- **Prose**: Text → Enhancement → Polished output
- **Prompt Enhancement**: Rough prompt → Refined prompt

### Report Styles
Configurable via API or UI:
- `ACADEMIC`: Detailed with citations
- `POPULAR_SCIENCE`: Accessible language
- `NEWS`: Journalistic format
- `SOCIAL_MEDIA`: Concise, shareable
- `STRATEGIC_INVESTMENT`: Business-focused analysis

## Testing Guidelines

### Python Testing
- Place tests in `tests/` directory (unit tests in `tests/unit/`, integration in `tests/integration/`)
- Use pytest fixtures for setup
- Minimum coverage: 25% (configured in `pyproject.toml`)
- Test both synchronous and asynchronous code with `pytest-asyncio`
- Mock external dependencies (LLM calls, search APIs, database connections)

### Frontend Testing
- Jest for unit/integration tests
- Place tests next to components or in `__tests__` directories
- Run with `pnpm test:run`

### Running Tests in Development
```bash
# Watch mode for quick iteration
pytest tests/unit/test_specific.py -v

# Run specific test function
pytest tests/integration/test_workflow.py::test_function_name -v

# Skip slow tests
pytest -m "not slow"
```

## Code Style and Quality

### Python
- **Formatter**: Ruff (line length 88, PEP 8 compliant)
- **Linter**: Ruff with auto-fix
- **Type hints**: Required for function signatures
- **Python version**: 3.12+

### TypeScript/JavaScript
- **Style**: ESLint + Prettier
- **Type safety**: Strict TypeScript
- **Functional components**: Prefer hooks over class components

### Pre-commit Hooks
Link the pre-commit hook for automatic checks:
```bash
chmod +x pre-commit
ln -s ../../pre-commit .git/hooks/pre-commit
```

## Docker Deployment

### Backend Only
```bash
docker build -t deer-flow-api .
docker run -d -t -p 127.0.0.1:8000:8000 --env-file .env --name deer-flow-api-app deer-flow-api
```

### Full Stack (Backend + Frontend)
```bash
docker compose build
docker compose up
```

**Security Warning**: In production, add authentication to the web interface and evaluate security implications of `ENABLE_MCP_SERVER_CONFIGURATION` and `ENABLE_PYTHON_REPL`.

## API Endpoints

FastAPI server (`src/server/app.py`):

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/chat/stream` | POST | Deep research with SSE streaming |
| `/api/tts` | POST | Text-to-speech conversion |
| `/api/podcast/generate` | POST | Generate podcast from content |
| `/api/ppt/generate` | POST | Generate PowerPoint |
| `/api/prose/enhance` | POST | Enhance prose |
| `/api/prompt-enhance` | POST | Refine prompts |
| `/api/rag/resources` | POST/GET | RAG resource management |
| `/api/mcp/servers` | GET/POST | MCP server configuration |
| `/api/evaluate` | POST | Report evaluation |

**SSE Response Format**: Events include `message_chunk`, `tool_calls`, `tool_call_result`, `interrupt`.

## Debugging with LangGraph Studio

LangGraph Studio provides visual debugging of multi-agent workflows:

1. Start the dev server: `make langgraph-dev`
2. Open Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
3. Submit a research topic and observe:
   - Graph visualization with node connections
   - Real-time execution traces
   - State inspection at each step
   - Input/output of each component
   - Provide feedback during planning phase

For LangSmith tracing, configure in `.env`:
```bash
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=xxx
LANGSMITH_PROJECT=xxx
```

## Common Development Patterns

### Working with State
```python
from src.graph.types import AgentState

def my_node(state: AgentState) -> dict:
    # Access state
    messages = state["messages"]
    plan = state.get("plan")

    # Return partial update (will be merged)
    return {"messages": [new_message]}
```

### Creating Tools
```python
from langchain_core.tools import tool

@tool
def my_tool(query: str) -> str:
    """Tool description for LLM."""
    # Implementation
    return result
```

### Error Handling
- Use try-except with specific exception types
- Log errors with context (use `logger` from `src/utils/`)
- Handle API failures gracefully with retries
- Provide meaningful error messages

### Async Operations
- Use `async`/`await` for I/O operations
- LangGraph nodes should be async functions
- Handle timeouts appropriately
- Clean up resources in `finally` blocks

## Entry Points

- **CLI**: `main.py` → `src/workflow.py:run_agent_workflow_async()`
- **API Server**: `server.py` → FastAPI app in `src/server/app.py`
- **Graph Building**: `src/graph/builder.py:build_graph_with_memory()`

## Acknowledgments

Built on top of:
- **LangChain**: LLM framework
- **LangGraph**: Multi-agent orchestration
- **Novel**: Notion-style editor for report editing
- **RAGFlow**: Private knowledge base integration

Core contributors: Daniel Walnut, Henry Li

# 智扫通机器人智能客服（SweepBot-Agent）

基于 **LangChain + LangGraph + 通义千问（DashScope）** 的扫地机器人智能客服系统。融合 **RAG 知识库检索** 与 **ReAct 智能体工具调用**，能够回答产品选购、故障排除、维护保养等专业问题，并能结合外部用户数据生成个性化使用报告。

## 功能特性

- **RAG 知识库问答**：将 `data/` 目录下的产品文档（txt / PDF）切分、向量化后存入 Chroma，回答问题时检索相关资料生成专业回答
- **ReAct 智能体**：基于 LangGraph 的自主思考循环，可自主决定调用工具获取信息（检索、天气、用户数据等）
- **动态提示词切换**：通过 middleware 在"普通问答"与"报告生成"两种场景间自动切换系统提示词
- **个性化使用报告**：根据用户 ID + 月份从外部记录（CSV）检索数据，生成机器使用情况报告与建议
- **工具调用监控**：middleware 记录每次工具调用的名称与参数，并维护运行时上下文
- **Streamlit 对话界面**：流式输出，打字机效果，支持多轮对话
- **MD5 文件去重**：已入库的文档通过 MD5 记录自动跳过，避免重复向量化
- **分层日志**：控制台 + 按天文件双输出（`logs/` 目录）

## 技术架构

```
用户提问
   │
   ▼
app.py (Streamlit 对话界面)
   │
   ▼
agent/react_agent.py (ReAct Agent, LangGraph)
   │  ├── middleware: log_before_model ──► 动态提示词 report_prompt_switch
   │  │                                       ├─ 普通问答 → prompts/main_prompt.txt
   │  │                                       └─ 报告场景 → prompts/report_prompt.txt
   │  ├── model: ChatTongyi (qwen3.7-max, DashScope)
   │  ├── tools:
   │  │    ├─ rag_summarize        → rag/rag_service.py → Chroma 向量检索
   │  │    ├─ get_weather          → 天气查询
   │  │    ├─ get_user_id          → 随机用户 ID
   │  │    ├─ get_user_location    → 随机城市
   │  │    ├─ get_current_month    → 当前月份
   │  │    ├─ fetch_external_data  → data/external/records.csv 用户使用记录
   │  │    └─ fill_context_for_report → 报告场景上下文标记（触发提示词切换）
   │  └── middleware: monitor_tool → 工具调用日志 + 运行时上下文维护
   │
   └── 回答流式返回
```

**报告生成执行流程**（由系统提示词约束）：`get_user_id` → `get_current_month` → `fill_context_for_report` → `fetch_external_data` → 生成报告

## 目录结构

```
SweepBot-Agent/
├── app.py                     # Streamlit 客服界面入口
├── agent/
│   ├── react_agent.py         # ReAct 智能体定义与流式执行
│   └── tools/
│       ├── agent_tools.py     # 工具集（RAG检索 / 天气 / 用户信息 / 外部数据）
│       └── middleware.py      # 中间件（工具监控 / 模型前日志 / 动态提示词）
├── rag/
│   ├── vector_store.py        # 向量库构建（txt/pdf → Chroma，MD5 去重）
│   └── rag_service.py         # RAG 总结服务（检索 + 生成链）
├── model/
│   └── factory.py             # 模型工厂（ChatTongyi / DashScopeEmbeddings）
├── utils/
│   ├── path_tool.py           # 工程绝对路径工具
│   ├── config_handler.py      # YAML 配置加载
│   ├── logger_handler.py      # 日志配置（控制台 + 文件）
│   ├── prompt_loader.py       # 提示词加载
│   └── file_handler.py        # 文件工具（MD5 / 目录扫描 / txt-pdf 加载）
├── config/
│   ├── rag.yml                # 模型配置（chat_model_name / embedding_model_name）
│   ├── chroma.yml             # 向量库配置（数据路径 / 分块 / 去重文件）
│   ├── prompts.yml            # 提示词文件路径
│   └── agent.yml              # 外部数据路径
├── prompts/
│   ├── main_prompt.txt        # 客服主系统提示词
│   ├── rag_summarize_prompt.txt  # RAG 总结提示词
│   └── report_prompt.txt      # 报告生成提示词
├── data/
│   ├── *.txt / *.pdf          # 知识库文档（选购指南、故障排除、维护保养等）
│   └── external/records.csv   # 用户使用记录（报告数据源）
├── logs/                      # 运行日志（按天生成）
└── md5.text                   # 已入库文档 MD5 记录（自动生成，勿手动修改）
```

## 环境要求

- Python 3.14+（项目在 3.14 环境开发验证）
- 可访问通义千问（DashScope）API
- 主要依赖：

| 依赖 | 版本（验证环境） |
|---|---|
| langchain | 1.3.14 |
| langgraph | 1.2.10 |
| langchain-community | 0.4.2 |
| langchain-chroma | 1.1.0 |
| chromadb | 1.5.9 |
| dashscope | 1.26.5 |
| pypdf | 6.14.2 |
| streamlit | 1.61.1 |
| PyYAML | 6.0.3 |

## 快速开始

### 1. 安装依赖

```bash
python -m venv .venv
.venv/Scripts/activate   # Windows；Linux/macOS 用 source .venv/bin/activate

pip install langchain langchain-community langchain-chroma langgraph chromadb dashscope pypdf pyyaml streamlit
```

> 若官方源无法访问，可指定国内镜像：`pip install ... -i https://pypi.tuna.tsinghua.edu.cn/simple`

### 2. 配置 API Key

在环境变量中配置 DashScope（阿里云百炼）密钥：

```bash
# Windows
set DASHSCOPE_API_KEY=sk-xxxx

# Linux / macOS
export DASHSCOPE_API_KEY=sk-xxxx
```

模型与向量模型可在 `config/rag.yml` 中修改（默认 `qwen3.7-max` / `text-embedding-v4`）。

### 3. 准备知识库数据

将产品文档放入 `data/` 目录（支持 `.txt` / `.pdf`），支持的文件类型在 `config/chroma.yml` 的 `allow_knowledge_file_type` 中配置。

构建 / 更新向量库：

```bash
python rag/vector_store.py
```

> 已入库的文件会记录 MD5（`md5.text`），重复运行自动跳过；新增或修改文档后重新运行即可增量入库。

### 4. 启动

**命令行测试智能体：**

```bash
python agent/react_agent.py
```

**启动 Web 对话界面：**

```bash
streamlit run app.py
```

浏览器访问 `http://localhost:8501` 即可开始对话。

## 配置说明

### `config/rag.yml` — 模型配置

```yaml
chat_model_name: qwen3.7-max        # 对话模型
embedding_model_name: text-embedding-v4  # 向量模型
```

### `config/chroma.yml` — 向量库配置

```yaml
collection_name: agent               # 向量集合名
persist_directory: chroma_db         # 向量库持久化目录
k: 3                                 # 检索返回条数
data_path: data                      # 知识库文档目录
md5_hex_store: md5.text              # 去重记录文件
allow_knowledge_file_type: ["txt", "pdf"]  # 允许的文档类型
chunk_size: 200                      # 分块大小
chunk_overlap: 20                    # 分块重叠
separator: ["\n\n", "\n", ".", "!", "?", "。", "", " "]  # 分块分隔符
```

### `config/prompts.yml` — 提示词路径

指定 `prompts/` 目录下三个提示词模板的文件路径。

### `config/agent.yml` — 外部数据

```yaml
external_data_path: data/external/records.csv  # 用户使用记录（报告数据源）
```

## 核心设计说明

### ReAct 智能体（`agent/react_agent.py`）

基于 `langchain.agents.create_agent` 构建，工具与中间件以列表注入：

- **工具**：`rag_summarize`、`get_weather`、`get_user_id`、`get_user_location`、`get_current_month`、`fetch_external_data`、`fill_context_for_report`
- **中间件**：
  - `monitor_tool`：包装所有工具调用，输出日志；当调用 `fill_context_for_report` 时将运行时上下文 `report` 置为 `True`
  - `log_before_model`：每次调用模型前输出消息条数日志
  - `report_prompt_switch`：根据上下文 `report` 标志动态选择"报告提示词"或"客服主提示词"

### 报告场景（提示词切换）

用户请求生成个人使用报告时，模型按约束依次调用工具，其中 `fill_context_for_report` 触发上下文标记 `report=True`，middleware 随即切换系统提示词为报告模板（`prompts/report_prompt.txt`），使后续回答转为报告格式。

### RAG 链路（`rag/vector_store.py` → `rag_service.py`）

文档 → MD5 去重 → 切分 → 向量化（DashScope）→ 存入 Chroma；回答时检索 Top-K 相关片段，交由总结提示词（`prompts/rag_summarize_prompt.txt`）生成基于资料的准确回答。

## 常见问题

**Q：运行报 `ValueError: Function must have a docstring if description not provided.`**
工具函数必须通过 `@tool(description="...")` 装饰，且提供 description 或 docstring。

**Q：txt 文档加载报 `UnicodeDecodeError: 'gbk' codec...`**
中文 Windows 下 `TextLoader` 默认按 GBK 解码，需显式指定 UTF-8：`TextLoader(filepath, encoding="utf-8")`（本项目已处理）。

**Q：`data` 目录找不到 / 报"不是文件夹"**
项目内所有路径基于工程根目录解析（`utils/path_tool.py`），请从工程根目录运行脚本，或确保 `config/chroma.yml` 中 `data_path` 为有效相对路径。

**Q：回答没有调用天气 / 外部数据工具**
模型自主决定工具调用顺序。若需强制流程，请在 `prompts/main_prompt.txt` 中强化对应的调用约束。

**Q：向量库分布在多个 `chroma_db` 目录**
`persist_directory` 为相对路径时，从不同工作目录运行会在各处新建向量库。建议统一从工程根目录运行。

## 日志

日志输出到控制台与 `logs/agent_YYYY-MM-DD.log`（按天切分），工具调用、模型调用、提示词切换均有记录，便于排查智能体行为。

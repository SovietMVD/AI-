# 阿尔法视频助手 — 使用指南

> 一个懂创作的 AI 智能客服助手，帮助短视频创作者解决从内容制作到平台运营的各种问题。

---

## 快速开始

### 环境要求

- Python 3.11+
- （可选）Anthropic API Key（不配置也能用 Demo 模式体验）

### 1. 克隆项目

将项目文件夹 `doukuai-cs-agent` 复制到本地。

### 2. 安装依赖

```bash
cd doukuai-cs-agent
python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

### 3. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env` 文件，填入配置（可选，Demo 模式不需要）：

```env
ANTHROPIC_API_KEY=sk-ant-xxx       # Anthropic API Key（Demo 模式不需要）
ANTHROPIC_MODEL=claude-sonnet-4-6  # 使用的模型
DATABASE_URL=sqlite+aiosqlite:///data.db
CHROMA_PERSIST_DIR=./chroma_data
HOST=0.0.0.0
PORT=8000
DEBUG=true
```

### 4. 初始化知识库

```bash
python -m src.scripts.init_kb
```

首次运行会自动下载 BAAI/bge-small-zh-v1.5 中文向量模型（约 100MB），并构建知识库索引。成功后会显示：

```
知识库初始化完成，加载 20 条 FAQ，共 131 个向量
```

### 5. 启动服务

```bash
python -m src.main
```

服务启动后访问：

| 地址 | 说明 |
|------|------|
| `http://localhost:8000` | API 服务 |
| `http://localhost:8000/health` | 健康检查 |
| `http://localhost:8000/docs` | 自动生成的 API 文档（Swagger） |
| `frontend/chat-test.html` | 浏览器直接打开即可体验对话 |

---

## 两种运行模式

### Demo 模式（默认，无需 API Key）

`frontend/chat-test.html` 默认连接 `http://localhost:8000/api/v1/chat/demo`，不需要任何 API Key。Demo 模式内置了自然对话引擎：

- **闲聊识别**：打招呼、感谢、告别等 6 类场景
- **意图分类**：8 大类意图（闲聊、视频创作、账号、内容运营、变现、社交、平台、其他）
- **知识库检索**：131 个向量索引，覆盖 20 个高频 FAQ
- **情绪安抚**：检测到负面情绪时先共情再给方案
- **转人工模拟**：排队 → 客服接入 → 人工对话

### 完整模式（需要 Anthropic API Key）

配置好 `.env` 中的 `ANTHROPIC_API_KEY` 后，将前端中的 API 地址改为：

```javascript
const API = 'http://localhost:8000/api/v1/chat/send';
```

即可使用 Claude LLM 驱动的完整 Agent 能力，包括：
- LLM 精准意图分类
- 自动工具调用（知识检索、用户查询、工单创建）
- 上下文感知的多轮对话
- 自动对话摘要生成

---

## API 端点参考

### 用户端

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/chat/send` | POST | 发送消息（SSE 流式返回，需 API Key） |
| `/api/v1/chat/demo` | POST | Demo 模式发送消息（无需 API Key） |
| `/api/v1/chat/transfer` | POST | 请求转人工 |
| `/api/v1/chat/history` | GET | 查询历史对话 `?conversation_id=xxx` |
| `/api/v1/chat/feedback` | POST | 提交满意度评价 |

**请求格式：**

```json
{
  "message": "新手怎么拍视频？",
  "user_id": "user_123",
  "user_type": "creator",
  "user_name": "小明",
  "conversation_id": ""
}
```

**响应格式（SSE 流式）：**

```
data: 你好呀~
data: 关于拍摄技巧，这里有一些建议...
event: done
data: demo_resolved
data: [DONE]
```

### 客服工作台

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/agent/sessions` | GET | 获取会话列表 `?status=pending&limit=20` |
| `/api/v1/agent/sessions/{id}/claim` | POST | 认领会话 `?agent_id=xxx` |

### 知识库管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/kb/search` | GET | 知识库检索 `?query=xxx&category=video_create&top_k=5` |
| `/api/v1/kb/entries` | POST | 批量导入知识条目 |

**批量导入示例：**

```json
{
  "entries": [
    {
      "question": "怎么提高视频清晰度？",
      "answer": "拍摄时注意光线充足，导出选择 1080P 以上分辨率...",
      "category": "video_create",
      "similar_questions": ["视频模糊怎么办", "画质差", "怎么拍高清"]
    }
  ]
}
```

---

## 项目结构

```
doukuai-cs-agent/
├── README.md
├── requirements.txt
├── .env.example
├── .cursorrules
├── frontend/
│   └── chat-test.html          # 用户端测试对话界面
├── knowledge_base/
│   └── sample_faqs.json        # FAQ 知识库（20 条，可按格式扩充）
├── src/
│   ├── main.py                 # 服务入口
│   ├── agent/
│   │   ├── core.py             # Agent 主循环（感知→检索→推理→行动）
│   │   ├── prompts.py          # 系统提示词模板
│   │   ├── intent.py           # 意图分类器（LLM + 关键词兜底）
│   │   ├── memory.py           # 分层记忆系统
│   │   └── tools.py            # 工具注册（知识检索、工单、转人工）
│   ├── rag/
│   │   ├── embeddings.py       # SentenceTransformer 向量化
│   │   ├── loader.py           # 知识库加载（JSON → Chroma）
│   │   └── retriever.py        # 向量检索 + 类别过滤
│   ├── api/
│   │   ├── routes.py           # FastAPI 路由（含 Demo 自然对话引擎）
│   │   └── middleware.py       # 敏感信息脱敏中间件
│   ├── db/
│   │   ├── models.py           # SQLAlchemy 数据模型
│   │   └── repository.py       # 数据访问层
│   ├── scripts/
│   │   └── init_kb.py          # 知识库初始化脚本
│   └── utils/
│       ├── config.py           # 环境变量配置
│       └── logger.py           # 结构化日志
├── tests/
└── deploy/
    ├── Dockerfile
    └── docker-compose.yml
```

---

## 如何扩充知识库

编辑 `knowledge_base/sample_faqs.json`，按以下格式添加新条目：

```json
{
  "id": "kb_021",
  "question": "问题标题（一条完整的问句）",
  "answer": "详细回答（建议 100-300 字，分点列出操作步骤）",
  "category": "video_create",
  "similar_questions": ["同义问题1", "同义问题2", "用户可能会怎么问"]
}
```

**可用分类：** `video_create`（视频创作）、`account`（账号）、`content`（内容运营）、`monetize`（变现）、`social`（社交）、`platform`（平台）

添加后重新初始化：

```bash
python -m src.scripts.init_kb
```

---

## Docker 部署

```bash
# 构建镜像
docker build -t alpha-video-assistant -f deploy/Dockerfile .

# 启动服务
docker-compose -f deploy/docker-compose.yml up -d
```

---

## 常用命令速查

```bash
# 初始化 / 更新知识库
python -m src.scripts.init_kb

# 启动服务（开发模式，代码改动自动重载）
python -m src.main

# 或使用 uvicorn 直接启动
uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload

# 运行测试
pytest tests/ -v

# 查看 API 文档
# 浏览器打开 http://localhost:8000/docs
```

---

## 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| Web 框架 | FastAPI | 原生异步 + SSE 流式 |
| LLM | Anthropic Claude Sonnet 4.6 | 中文对话 + 工具调用 |
| 向量模型 | BAAI/bge-small-zh-v1.5 | 512 维中文 Embedding |
| 向量数据库 | Chroma | 开发环境嵌入式 |
| 关系数据库 | SQLite (开发) / PostgreSQL (生产) | SQLAlchemy 2.0 Async |
| 缓存 | Redis | 高频问答缓存 |
| 日志 | structlog | 结构化 JSON 日志 |

---

## 常见问题

**Q: 启动报错 `ModuleNotFoundError`？**
A: 确保已激活虚拟环境并执行 `pip install -r requirements.txt`。

**Q: Demo 模式回复不够智能？**
A: Demo 模式基于关键词匹配 + 知识库检索，不需要 LLM。配置 Anthropic API Key 并切换到 `/api/v1/chat/send` 端点可获得 LLM 驱动的完整 Agent 体验。

**Q: 知识库初始化很慢？**
A: 首次运行需要从 HuggingFace 下载 BGE 向量模型（约 100MB），之后会使用缓存。

**Q: Windows 终端显示乱码？**
A: Windows 默认 GBK 编码不支持 emoji。在代码中已处理 UTF-8 输出，浏览器前端不受影响。

**Q: 如何切换 LLM 提供商？**
A: 修改 `src/api/routes.py` 中的 `anthropic_client` 初始化，替换为 OpenAI / DeepSeek 等兼容 SDK。

---

## 许可证

MIT License

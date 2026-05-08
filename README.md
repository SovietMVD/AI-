# 阿尔法 AI 智能客服

基于 Claude API 的短视频平台 AI 客服系统，支持智能问答、意图识别、知识库检索、人工转接。

## 快速开始

```bash
# 1. 创建虚拟环境
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env，填入 ANTHROPIC_API_KEY

# 4. 初始化知识库
python -m src.scripts.init_kb

# 5. 启动服务
python -m src.main
# 或：uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
```

## API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/chat/send` | POST | 发送消息（SSE 流式返回） |
| `/api/v1/chat/transfer` | POST | 转人工 |
| `/api/v1/chat/history` | GET | 查询历史对话 |
| `/api/v1/chat/feedback` | POST | 提交满意度评价 |
| `/api/v1/kb/search` | GET | 知识库检索 |
| `/api/v1/kb/entries` | POST | 批量导入知识 |
| `/api/v1/agent/sessions` | GET | 客服会话列表 |
| `/health` | GET | 健康检查 |

## 项目结构

```
src/
├── main.py              # 服务入口
├── agent/
│   ├── core.py           # Agent 主循环（感知→检索→推理→行动）
│   ├── prompts.py        # 系统提示词模板
│   ├── intent.py         # 意图分类器（LLM + 关键词兜底）
│   ├── memory.py         # 分层记忆（工作 + 短期 + 长期）
│   └── tools.py          # 工具注册（知识检索、工单、转人工）
├── rag/
│   ├── embeddings.py     # SentenceTransformer 向量化
│   ├── loader.py         # 知识库加载（JSON → Chroma）
│   └── retriever.py      # 向量检索 + 类别过滤
├── api/
│   ├── routes.py         # FastAPI 路由
│   └── middleware.py     # 敏感信息脱敏
├── db/
│   ├── models.py         # SQLAlchemy 数据模型
│   └── repository.py     # 数据访问层
└── utils/
    ├── config.py         # 配置管理
    └── logger.py         # 结构化日志
```

## 运行测试

```bash
pytest tests/ -v
```

## Docker 部署

```bash
docker-compose -f deploy/docker-compose.yml up -d
```

## 技术栈

- **LLM**：Anthropic Claude Sonnet 4.6
- **Web**：FastAPI + SSE 流式
- **向量库**：Chroma（开发）/ Milvus（生产）
- **Embedding**：BAAI/bge-small-zh-v1.5（384维）
- **数据库**：SQLite（开发）/ PostgreSQL（生产）
- **缓存**：Redis

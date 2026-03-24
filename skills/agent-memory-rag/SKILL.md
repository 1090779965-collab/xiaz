---
name: agent-memory-rag
description: |
  Agent记忆与RAG检索Skill。构建具备长期记忆的智能Agent，结合向量检索实现知识复用。
  适用于：(1) 个性化助手 (2) 聊天机器人 (3) 知识密集型任务 (4) 
  用户说"记住之前说的"、"知识库"、"向量检索"等场景。
---

# Agent记忆与RAG检索

## 核心概念

### RAG vs 记忆系统

| 特性 | RAG | 记忆系统 |
|------|-----|----------|
| 定位 | 静态外部知识检索 | 动态状态积累 |
| 状态 | 无状态 | 有状态 |
| 更新 | 手动更新 | 自动学习 |
| 成本 | 边际成本稳定 | 复用降低边际成本 |

**最佳实践**：RAG + 记忆系统协同

## 分层记忆架构

### 三层记忆

```
┌─────────────────────────────────┐
│     长期记忆（结构化数据库）      │  ← 用户画像、历史订单、关系数据
├─────────────────────────────────┤
│     中期记忆（向量数据库）        │  ← 语义检索、兴趣偏好
├─────────────────────────────────┤
│     短期记忆（上下文窗口）        │  ← 当前会话、任务进度
└─────────────────────────────────┘
```

### 各层职责

| 层级 | 存储内容 | 检索方式 | 容量 |
|------|----------|----------|------|
| 短期 | 会话上下文、当前任务状态 | 直接读取 | ~128k tokens |
| 中期 | 关键信息摘要、偏好 | 向量相似度 | 无限制 |
| 长期 | 结构化数据、档案 | SQL查询 | 无限制 |

## RAG检索流程

### 标准RAG

```
用户查询 → 向量化 → 向量数据库检索 → Top-K结果 → 拼接Prompt → LLM生成
```

### 关键组件

1. **向量化模型**：text-embedding-ada-002, BGE, etc.
2. **向量数据库**：Pinecone, Milvus, FAISS, Qdrant
3. **分块策略**：固定大小、语义分割、递归分块
4. **检索排序**：相似度、重排序、多路召回

## 记忆更新机制

### 更新策略

| 策略 | 触发条件 | 适用场景 |
|------|----------|----------|
| 全部存储 | 每次对话 | 数据量小 |
| 选择性存储 | 重要信息 | 数据量大 |
| 摘要存储 | 会话结束 | 长期记忆 |

### 信息衰减

```
重要性 = f(出现频率, 时间衰减, 用户确认)
```

- 频繁出现 → 重要性↑
- 时间久远 → 重要性↓
- 用户确认 → 重要性↑↑

## 实现示例

### 1. 短期记忆（会话上下文）

```python
# 简单实现：维护会话历史
class ShortTermMemory:
    def __init__(self, max_tokens=32000):
        self.messages = []
        self.max_tokens = max_tokens
    
    def add(self, role, content):
        self.messages.append({"role": role, "content": content})
        self._truncate()
    
    def get_context(self):
        return "\n".join([f"{m['role']}: {m['content']}" 
                         for m in self.messages])
```

### 2. 中期记忆（向量检索）

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

# 构建向量存储
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings()
)

# 检索
def retrieve(query, k=3):
    return vectorstore.similarity_search(query, k=k)
```

### 3. 长期记忆（结构化存储）

```python
# PostgreSQL + pgvector
class LongTermMemory:
    def __init__(self):
        self.db = psycopg2.connect(...)
    
    def store_user_profile(self, user_id, profile):
        # 存储用户画像
        sql = "INSERT INTO user_profiles VALUES (%s, %s)"
        self.db.execute(sql, (user_id, json.dumps(profile)))
    
    def get_user_preferences(self, user_id):
        sql = "SELECT preferences FROM user_profiles WHERE id = %s"
        return self.db.execute(sql, (user_id,)).fetchone()
```

## Agentic RAG（2025进阶）

### 核心思想

传统RAG：被动检索
Agentic RAG：检索 + 反思 + 记忆

```
用户查询 → 理解意图 → 判断是否检索 → 检索 → 反思结果 → 生成 → 更新记忆
```

### 实现框架

```python
class AgenticRAG:
    def __init__(self):
        self.memory = MemorySystem()
        self.retriever = Retriever()
        self.llm = LLM()
    
    def chat(self, query):
        # 1. 检查记忆
        context = self.memory.recall(query)
        
        # 2. 判断是否需要检索
        if self.need_retrieve(query, context):
            # 3. 检索
            docs = self.retriever.search(query)
            context += "\n" + docs
        
        # 4. 生成
        response = self.llm.generate(query, context)
        
        # 5. 更新记忆
        self.memory.remember(query, response)
        
        return response
    
    def need_retrieve(self, query, context):
        # 判断是否需要外部检索
        if "最新" in query or "实时" in query:
            return True
        if not context:
            return True
        return False
```

## 工具推荐

| 工具 | 用途 |
|------|------|
| LangChain | Agent框架 |
| LangGraph | 状态管理 |
| MemGPT | 长期记忆管理 |
| LlamaIndex | RAG专用 |
| Chroma | 本地向量数据库 |
| Pinecone | 云端向量服务 |

## 适用场景

### 适合使用记忆系统
- ✅ 个性化聊天（记住用户偏好）
- ✅ 多轮对话（保持上下文）
- ✅ 知识管理（构建知识库）
- ✅ 任务助手（记住任务进度）

### 适合使用RAG
- ✅ 文档问答
- ✅ 知识查询
- ✅ 信息检索

### 最佳组合
- Agentic RAG：检索+反思+记忆
- 场景：客服机器人、个人助手、知识库

## 实践检查清单

### 记忆系统
- [ ] 短期记忆容量足够
- [ ] 中期记忆检索准确
- [ ] 长期记忆结构清晰
- [ ] 隐私合规（脱敏）

### RAG
- [ ] 分块大小合适
- [ ] 向量化模型匹配
- [ ] Top-K设置合理
- [ ] 检索结果相关

### 整体
- [ ] 延迟可接受
- [ ] 成本可控
- [ ] 效果达预期

---
name: rag-practice
description: |
  RAG实践Skill。构建企业知识库和问答系统的主流技术。
  适用于：(1) 知识库问答 (2) 文档检索 (3) 企业内部知识管理 (4)
  用户说"RAG"、"知识库"、"向量检索"、"文档问答"等场景。
---

# RAG实践

## 核心架构

```
用户查询 → 向量化 → 向量数据库检索 → Top-K结果 → 拼接Prompt → LLM生成
```

## 核心组件

### 1. 文档加载

```python
from langchain.document_loaders import PyPDFLoader, TextLoader

# PDF
loader = PyPDFLoader("document.pdf")
pages = loader.load()

# 文本
loader = TextLoader("document.txt")
docs = loader.load()
```

### 2. 文本分块

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # 块大小
    chunk_overlap=50,    # 重叠
    separators=["\n", "。", " "]
)

chunks = splitter.split_documents(docs)
```

### 3. 向量化

```python
from langchain.embeddings import OpenAIEmbeddings

# OpenAI
embeddings = OpenAIEmbeddings(model="text-embedding-ada-002")

# 本地
# from langchain.embeddings import HuggingFaceEmbeddings
# embeddings = HuggingFaceEmbeddings(model="BAAI/bge-small-zh")
```

### 4. 向量存储

```python
from langchain.vectorstores import Chroma

# 本地存储
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)
```

### 5. 检索

```python
# 相似度检索
docs = vectorstore.similarity_search(query, k=3)

# 带分数
docs_scores = vectorstore.similarity_search_with_score(query, k=3)
```

### 6. 生成

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import PromptTemplate

llm = ChatOpenAI(temperature=0)

# 构建Prompt
prompt = PromptTemplate.from_template("""
基于以下上下文回答问题：

上下文：
{context}

问题：{question}

回答：
""")

# 链式调用
chain = prompt | llm
response = chain.invoke({
    "context": "\n\n".join([d.page_content for d in docs]),
    "question": query
})
```

## 分块策略

### 固定大小分块

```python
from langchain.text_splitter import CharacterTextSplitter

splitter = CharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=0
)
```

### 递归分块

```python
# 按段落、句子分，更智能
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", " ", ""]
)
```

### 语义分块（高级）

```python
# 用LLM判断语义边界
from langchain.experimental.semantic_chunk import SemanticChunker

semantic_splitter = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile"
)
```

### 分块大小选择

| 场景 | 块大小 | 重叠 |
|------|--------|------|
| 短问答 | 200-300 | 50 |
| 文档问答 | 500-1000 | 100 |
| 摘要提取 | 1000-2000 | 200 |

## 向量数据库

### 选择指南

| 数据库 | 特点 | 适用场景 |
|--------|------|----------|
| Chroma | 轻量、本地 | 开发测试 |
| Pinecone | 云服务、易用 | 中小规模 |
| Milvus | 国产、高性能 | 大规模 |
| Weaviate | 图形化、灵活 | 复杂查询 |
| Qdrant | 性能高 | 实时检索 |

### Milvus示例

```python
from pymilvus import connections, Collection

# 连接
connections.connect("default", host="localhost", port="19530")

# 搜索
search_params = {"metric_type": "IP", "params": {}}
results = collection.search(
    vectors=[query_vector],
    anns_field="vector",
    param=search_params,
    limit=10
)
```

## 检索优化

### 1. Multi-Query

```python
from langchain.retrievers import MultiQueryRetriever

retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=llm
)
```

### 2. Contextual Compression

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(llm)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=retriever
)
```

### 3. Re-Ranking

```python
from langchain.retrievers import ContextualCompressionRetriever

# 第一轮：粗筛
docs = vectorstore.similarity_search(query, k=20)

# 第二轮：精筛（用LLM重排）
reranked = llm.rerank(documents=docs, query=query, top_n=5)
```

## RAG工作流

### 基础RAG

```
Query → Embed → Search → Get Context → Generate
```

### 高级RAG

```
Query → Rewrite → Embed → Search → Get Context → Rerank → Generate
```

### Agentic RAG

```
Query → Check Memory → (if needed) Search → Update Memory → Generate
```

## 评估与优化

### 评估指标

| 指标 | 说明 |
|------|------|
| Precision@K | 召回的K个中相关的比例 |
| Recall@K | 相关文档召回的比例 |
| MRR | 第一个相关文档的排名倒数 |
| NDCG | 标准化折损累计增益 |

### 常见问题

| 问题 | 解决 |
|------|------|
| 召回不相关 | 优化chunk size/overlap |
| 遗漏关键信息 | 增加重叠、调整top_k |
| 生成幻觉 | 增加context质量 |
| 回答太长 | 限制输出长度 |

## 生产部署

### 1. 监控

```python
# 记录query和response
def log_rag_interaction(query, response, context_docs):
    log = {
        "query": query,
        "response": response,
        "sources": [d.metadata for d in context_docs],
        "timestamp": datetime.now()
    }
    # 存储到日志系统
```

### 2. 缓存

```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_embed(text):
    return embeddings.embed_query(text)
```

### 3. 增量更新

```python
def update_knowledge_base(new_documents):
    # 新文档分块
    new_chunks = splitter.split_documents(new_documents)
    # 增量添加到向量库
    vectorstore.add_documents(new_chunks)
```

## 实践检查清单

- [ ] 文档加载正常
- [ ] 分块大小合适
- [ ] 向量化模型匹配
- [ ] 检索结果相关
- [ ] 生成回答准确
- [ ] 延迟可接受
- [ ] 成本可控

## 工具推荐

| 工具 | 用途 |
|------|------|
| LangChain | RAG框架 |
| LlamaIndex | RAG专用 |
| LangFlow | 可视化RAG |
| RAGAS | RAG评估 |

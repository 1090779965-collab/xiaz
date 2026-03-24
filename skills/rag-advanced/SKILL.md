# 🧠 rag-advanced - RAG高级实践

## 1. 技能概述

**核心能力**：构建企业级知识检索增强系统，支持大规模文档、精准召回、多轮对话

**适用场景**：
- 企业内部知识库问答
- 客服机器人+产品文档
- 论文/报告智能分析
- 法律/财务文档检索

---

## 2. 核心技术

### 2.1 RAG架构演进

```
V1.0: 朴素RAG
  文档 → 分块 → 向量 → 相似检索 → LLM生成

V2.0: 优化RAG
  文档 → 分块(重叠) → 向量 → 层级索引 → rerank → LLM生成

V3.0: Agentic RAG
  查询理解 → 多路检索 → 工具调用 → 动态知识 → LLM推理 → 生成
```

### 2.2 核心组件

| 组件 | 工具选择 | 作用 |
|------|----------|------|
| 文档加载 | LangChain/Loaders | PDF/Word/MD解析 |
| 文本分块 | RecursiveCharacterTextSplitter | 语义完整分块 |
| 向量存储 | Chroma/Milvus/Qdrant | 高效相似检索 |
| 检索器 | BM25+向量混合 | 多路召回 |
| 重排序 | CrossEncoder | 精准排序 |
| 生成 | GPT-4/Claude | 答案生成 |

### 2.3 分块策略

```python
# 最佳实践：重叠分块
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # 块大小
    chunk_overlap=50,    # 重叠50字符，保持语义连贯
    separators=["\n\n", "\n", "。", ".", " "]
)
```

---

## 3. 高级技巧

### 3.1 查询理解

```
用户输入 → 意图识别 → 关键词提取 → 查询改写

示例：
原始：我想了解一下你们公司有什么产品
意图：产品咨询
关键词：产品、类型、分类
改写：公司产品分类列表
```

### 3.2 多路召回

```python
# 融合：向量检索 + BM25 + GPU
results = {
    "vector": vector_retriever.invoke(query),  # 语义相似
    "keyword": bm25_retriever.invoke(query),   # 关键词精确
    "graph": kg_retriever.invoke(query)        # 知识图谱
}
# 归一化分数后融合排序
```

### 3.3 重排序（Rerank）

```python
# 使用CrossEncoder精排
from sentence_transformers import CrossEncoder
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

reranked = reranker.rank(query, candidates)
```

### 3.4 主动学习

```
发现不确定答案 → 反问用户确认 → 更新知识库
用户反馈错误 → 标记问题 → 人工/自动修正
```

---

## 4. 实际案例

### 案例：企业知识库问答

```
需求：员工手册、产品文档、技术FAQ
文档量：500+份PDF/Word
日活：1000+提问

架构：
1. 文档预处理（OCR/解析）
2. 分块+向量化（夜间增量更新）
3. 查询路由（判断是否需要检索）
4. 混合检索（向量+关键词）
5. Rerank + Context压缩
6. LLM生成（带引用）
```

**效果**：
- 首次召回率：78% → 94%（加Rerank）
- 平均响应时间：2.3秒
- 用户满意度：4.6/5

---

## 5. 性能优化

### 5.1 检索优化

| 策略 | 效果 |
|------|------|
| 批量向量化 | 吞吐量提升10x |
| 缓存热门查询 | 延迟降低60% |
| 异步索引 | 索引不阻塞查询 |
| 分层索引 | 大文档检索不遗漏 |

### 5.2 生成优化

```
Context压缩：只保留相关段落（StripMiner）
引用来源：生成时附带原文引用
拒绝回答：低置信度时转人工
```

---

## 6. 常见问题

**Q：文档太大内存放不下？**
A：用分层索引（上层摘要+下层详情），按需加载

**Q：召回不准确？**
A：检查分块是否合理，增加重叠；尝试不同Embedding模型

**Q：生成幻觉？**
A：强制引用原文，开启RAG+Self-Check

---

## 7. 实战练习

**任务1**：搭建简易RAG
```
用LangChain + Chroma搭建PDF问答系统
```

**任务2**：优化检索
```
添加BM25混合检索 + Rerank
```

**任务3**：生产级部署
```
添加查询缓存 + 异步索引 + 监控
```

---

## 8. 进阶方向

- [ ] Graph RAG：知识图谱增强
- [ ] Agentic RAG：自主判断检索时机
- [ ] Multi-modal RAG：支持图片/视频检索
- [ ] 个性化RAG：用户画像+偏好学习

---

*🦞 Skill by 虾米 - 让知识检索更智能*

---
name: context-extension
description: |
  上下文扩展Skill。通过RoPE位置编码等技术突破模型上下文限制，处理超长文本。
  适用于：(1) 长文档分析 (2) 法律合同审查 (3) 代码库理解 (4) 
  用户说"长文本"、"上下文窗口"、"超长文档"等场景。
---

# 上下文扩展

## 核心问题

Transformer的**O(n²)**复杂度限制：
- 注意力计算随序列长度平方增长
- 显存随序列长度线性增长

**挑战**：如何在有限资源下处理更长文本？

## 核心技术

### 1. RoPE旋转位置编码

#### 核心思想

通过**旋转矩阵**将位置信息嵌入词向量：

```python
import torch
import math

def rotate_half(x):
    """旋转半个向量"""
    x1, x2 = x[..., ::2], x[..., 1::2]
    return torch.cat([-x2, x1], dim=-1)

def apply_rotary_pos_emb(q, k, position_ids):
    """应用RoPE"""
    # 频率计算：θ_d = 10000^(-2d/D)
    dim = q.shape[-1]
    inv_freq = 1.0 / (10000 ** (torch.arange(0, dim, 2).float() / dim))
    
    # 位置角
    t = torch.outer(position_ids.float(), inv_freq)
    
    # 旋转
    cos = torch.cos(t)
    sin = torch.sin(t)
    
    q_rot = q * cos + rotate_half(q) * sin
    k_rot = k * cos + rotate_half(k) * sin
    
    return q_rot, k_rot
```

#### 优势

- ✅ **相对位置建模**：注意力依赖相对距离
- ✅ **外推性强**：可处理未训练过的长度
- ✅ **计算高效**：旋转操作可并行

### 2. 位置编码扩展方法

| 方法 | 原理 | 效果 |
|------|------|------|
| 线性缩放 | 修改位置索引 | 简单，效果有限 |
| NTK-aware | 调整频率基数 | 外推效果好 |
| YaRN | 动态频率缩放 | 长文本稳定 |
| PI | 位置插值 | 平滑过渡 |

```python
# NTK-aware扩展
def ntk_aware_scaled_rotary_embedding(
    original_seq_len, 
    target_seq_len, 
    dim
):
    # 计算缩放因子
    scale = (target_seq_len / original_seq_len) ** (dim / (dim - 2))
    # 调整频率
    inv_freq = 1.0 / (scale * (10000 ** (torch.arange(0, dim, 2).float() / dim)))
    ...
```

### 3. 稀疏注意力

#### 核心思想

只计算关键Token的注意力：

```python
# 滑动窗口注意力
class SlidingWindowAttention(nn.Module):
    def __init__(self, window_size=512):
        self.window_size = window_size
    
    def forward(self, q, k, v):
        # 只计算窗口内的注意力
        seq_len = q.shape[1]
        mask = torch.tril(torch.ones(seq_len, seq_len))
        mask = mask < self.window_size
        return scaled_dot_product_attention(q, k, v, attn_mask=mask)
```

#### 稀疏模式

| 模式 | 描述 | 复杂度 |
|------|------|--------|
| 滑动窗口 | 只看相邻Token | O(n×w) |
| 稀疏全局 | 部分全局+局部 | O(n×√n) |
| 块状 | 分块计算 | O(n²/b²) |
| 随机 | 随机采样 | O(n×k) |

### 4. RingAttention（2025突破）

#### 核心思想

分布式环形计算，突破单机内存：

```
设备0: KV[0-1000]  →  设备1: KV[1001-2000]
↑                              ↓
设备3: KV[7001-8000] ← 设备2: KV[6001-7000]
```

```python
# 伪代码
def ring_attention(q, k_chunks, v_chunks):
    outputs = []
    for i in range(len(k_chunks)):
        # 聚合KV
        k_accum = k_chunks[0]
        v_accum = v_chunks[0]
        
        # 环形通信
        for j in range(1, num_devices):
            k_accum = ring_send_receive(k_accum, k_chunks[j])
            v_accum = ring_send_receive(v_accum, v_chunks[j])
            
            # 计算注意力
            attn = attention(q, k_accum, v_accum)
            outputs.append(attn)
    
    return concat(outputs)
```

### 5. KV Cache压缩

#### 核心思想

压缩键值对存储：

```python
# 量化压缩
def quantize_kv(kv_cache, bits=4):
    # 聚类量化
    codebook, indices = kmeans(kv_cache, num_clusters=2**bits)
    # 解码时查表
    return codebook[indices]

# 滑动窗口压缩
def compress_kv(kv_cache, window_size=512):
    # 只保留最近window_size个
    return kv_cache[:, -window_size:, :]
```

## 商业产品进展

| 产品 | 上下文长度 | 技术 |
|------|-----------|------|
| Claude 4 | 100万Token | 改进RoPE+稀疏注意 |
| Gemini Ultra | 200万Token | RingAttention |
| NVIDIA | 400万Token | 两阶段训练 |
| Kimi | 200万Token | RoPE扩展 |

## 处理长文本策略

### 策略1：直接处理
适用于：模型原生支持长上下文

```python
# 直接输入
response = llm.generate(long_document, max_tokens=1000)
```

### 策略2：分块处理
适用于：超出模型限制

```python
def process_long_text(text, chunk_size=8000):
    chunks = split_into_chunks(text, chunk_size)
    results = []
    for chunk in chunks:
        result = llm.generate(chunk)
        results.append(result)
    return summarize(results)
```

### 策略3：层次化处理
适用于：超长文档（10万+Token）

```摘要 → 详细内容 → 问答

```python
# L1: 生成摘要
summary = llm.generate(document, prompt="生成摘要")

# L2: 章节概要
sections = split_by_headings(document)
section_summaries = [llm.generate(s) for s in sections]

# L3: 问答
answer = llm.generate(summary + sections, question)
```

## 实践检查清单

### 模型选择
- [ ] 模型原生支持长度
- [ ] 扩展后效果验证
- [ ] 推理延迟可接受

### 文本处理
- [ ] 分块策略合理
- [ ] 块间重叠
- [ ] 信息不丢失

### 效果评估
- [ ] 关键信息检索准确
- [ ] 长距离依赖保留
- [ ] 无"记忆稀释"

## 适用场景

### 适合长上下文
- ✅ 法律合同审查
- ✅ 医疗病历分析
- ✅ 代码库理解
- ✅ 书籍/报告总结

### 替代方案
- ❌ 简单问答 → 直接短文本
- ❌ 实时对话 → RAG更合适
- ❌ 资源受限 → 分块+RAG

## 工具推荐

| 工具 | 用途 |
|------|------|
| Transformers | RoPE实现 |
| Flash Attention | 高效注意力 |
| vLLM | 长上下文推理 |
| LangChain | 长文本处理 |

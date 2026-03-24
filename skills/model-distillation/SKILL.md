---
name: model-distillation
description: |
  模型蒸馏Skill。将大模型压缩为小模型，保留性能同时降低推理成本。
  适用于：(1) 边缘设备部署 (2) 降低推理延迟 (3) 成本优化 (4) 
  用户说"模型压缩"、"蒸馏"、"学生模型"等场景。
---

# 模型蒸馏

## 核心原理

### 什么是蒸馏

Teacher-Student知识迁移：
- **教师模型**：大而强（参数多，能力强）
- **学生模型**：小而巧（参数少，部署易）

核心：学生不仅学习**硬标签**，还学习教师的**软标签**

### 知识类型

| 知识类型 | 描述 | 蒸馏位置 |
|----------|------|----------|
| 输出层知识 | 软标签概率分布 | 最后一层 |
| 中间特征 | 隐藏层表示 | 中间层 |
| 关系知识 | 样本间关系 | 关系矩阵 |

## 核心技术

### 1. 温度参数（Temperature）

控制软标签平滑度：

```python
def soft_label(logits, temperature=1.0):
    return F.softmax(logits / temperature, dim=-1)
```

- **T=1**：标准Softmax
- **T>1**：分布更平滑，保留类别相似信息

### 2. 损失函数

```python
# 蒸馏损失 = 软损失 + 硬损失
loss = α * KL_divergence(student_soft, teacher_soft) + 
       (1-α) * CrossEntropy(student_logits, labels)

# KL散度
L_soft = KL(student_soft || teacher_soft) 
       = Σ P_teacher * log(P_teacher / P_student)

# 交叉熵
L_hard = CE(student_logits, labels)
```

### 3. 蒸馏策略

| 策略 | 描述 | 适用场景 |
|------|------|----------|
| 离线蒸馏 | 教师固定，学生学习 | 通用压缩 |
| online蒸馏 | 教师学生共同训练 | 动态场景 |
| 自蒸馏 | 自己教自己 | 知识增强 |
| 多教师 | 多个教师教一个学生 | 知识融合 |

## 执行流程

### 1. 准备阶段

```python
# 加载教师模型
teacher = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2-72B")
teacher.eval()

# 初始化学生模型
student = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2-7B")
```

### 2. 蒸馏训练

```python
from torch.nn import functional as F

def distillation_loss(student_logits, teacher_logits, labels, 
                     temperature=4.0, alpha=0.7):
    # 软损失
    soft_student = F.log_softmax(student_logits / temperature, dim=-1)
    soft_teacher = F.softmax(teacher_logits / temperature, dim=-1)
    L_soft = F.kl_div(soft_student, soft_teacher, reduction='batchmean')
    L_soft = L_soft * (temperature ** 2)  # 温度补偿
    
    # 硬损失
    L_hard = F.cross_entropy(student_logits, labels)
    
    # 加权组合
    return alpha * L_soft + (1 - alpha) * L_hard
```

### 3. 推理部署

```python
# 使用学生模型推理
student.generate(input_ids, max_new_tokens=100)
```

## 进阶技术（2025）

### 课程蒸馏

分阶段教学：
1. 先训练基础能力
2. 再学习复杂推理
3. 逐步增加难度

### 超图感知蒸馏

理论突破：在特定条件下，学生**可以超越**教师

```python
# 核心：动态Top-K稀疏化
# 关键：K值大于临界值
```

### 习惯推理蒸馏

把推理过程**内化**为模型参数：

```python
# 之前：推理时输出Token
# 现在：推理过程"融化"到权重中
# 效果：减少推理时的Token输出，降低延迟
```

### 多模态蒸馏

跨模态知识迁移：
- 视觉→语言
- 音频→语言
- 示例：LLaVA-MoD

## 超参数建议

| 参数 | 建议值 | 说明 |
|------|--------|------|
| temperature | 2-10 | 简单任务用小值 |
| alpha | 0.5-0.9 | 软损失权重 |
| batch_size | 16-64 | 根据显存调整 |
| learning_rate | 1e-4 ~ 1e-5 | 比全量训练小 |

## 训练技巧

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 学生效果差 | 教师太强/学生太弱 | 选择相近架构 |
| 过拟合 | 数据不足 | 数据增强/正则 |
| 收敛慢 | 学习率不当 | 预热+余弦衰减 |

### 优化技巧

1. **渐进式蒸馏**：先小温度，再大温度
2. **中间层对齐**：同时蒸馏中间层特征
3. **关系蒸馏**：学习样本间相似度
4. **数据增强**：扩充训练数据

## 工具推荐

| 工具 | 用途 |
|------|------|
| Transformers | 模型加载 |
| DistilBERT | 官方蒸馏模型 |
| OpenMatch | 蒸馏训练框架 |
| TML | 知识迁移库 |

## 适用场景

### 适合蒸馏
- ✅ 边缘设备部署（手机/嵌入式）
- ✅ 降低推理成本
- ✅ 加速模型响应
- ✅ 多租户服务

### 不适合蒸馏
- ❌ 学生模型架构差异大
- ❌ 数据量极小
- ❌ 任务差异大

## 性能基准参考

| 教师模型 | 学生模型 | 性能保持 | 加速比 |
|----------|----------|----------|--------|
| GPT-4 | GPT-3.5 | ~95% | 2-3x |
| 72B | 7B | ~90% | 5-10x |
| BERT-Base | BERT-Mini | ~85% | 10x+ |

## 实践检查清单

- [ ] 教师-学生架构兼容
- [ ] 温度参数合适
- [ ] 软硬损失权重平衡
- [ ] 数据增强充分
- [ ] 评估指标合理
- [ ] 部署环境适配

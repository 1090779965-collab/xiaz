---
name: lora-finetune
description: |
  LoRA高效微调Skill。通过低秩矩阵实现参数高效微调，在有限资源下微调大模型。
  适用于：(1) 领域适配（医疗、法律、金融等）(2) 特定任务微调 (3) 快速模型迭代 (4) 
  用户说"微调模型"、"训练自己的模型"、"LoRA"等场景。
---

# LoRA高效微调

## 核心原理

LoRA（Low-Rank Adaptation）核心思想：
- **冻结**预训练模型的原始权重
- 通过训练**低秩矩阵**来模拟权重更新
- 训练参数量减少**99%+**

数学公式：
```
ΔW = A × B  （其中 r << d）
W' = W + ΔW
```

## 关键参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| **rank (r)** | 8-32 | 简单任务用小值，复杂任务用大值 |
| **alpha (α)** | 2r | 缩放系数，调节更新影响 |
| **target modules** | q_proj, v_proj | 优先微调Attention层 |

## 执行流程

### 1. 准备数据

**数据格式**（JSONL）：
```json
{"instruction": "你是医疗助手", "input": "患者发烧38度", "output": "建议先进行物理降温..."}
```

**数据量建议**：
- 简单任务：100-1000条
- 复杂任务：1000-10000条
- 原则：**质量 > 数量**

### 2. 配置LoRA

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,                    # rank
    lora_alpha=32,           # alpha = 2 * r
    target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)
```

### 3. 训练

```bash
# 使用LLaMA-Factory
llamafactory-cli train \
    --model_name_or_path Qwen/Qwen2-7B \
    --dataset my_data \
    --lora_rank 16 \
    --lora_alpha 32 \
    --output_dir ./output
```

### 4. 合并与推理

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM

# 加载基座模型
base = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2-7B")
# 加载LoRA权重
model = PeftModel.from_pretrained(base, "./output")
# 合并
merged = model.merge_and_unload()
# 推理
merged.generate(...)
```

## QLoRA（更省资源）

在LoRA基础上引入**量化**，单卡可微调65B模型：

```python
from transformers import BitsAndBytesConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype="float16"
)
```

## 训练技巧

### 超参数建议

| 参数 | 建议值 |
|------|--------|
| learning_rate | 1e-4 ~ 5e-4 |
| num_epochs | 3-5 |
| batch_size | 1-4 |
| gradient_accumulation | 4-16 |

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 过拟合 | 降低r值，增加dropout |
| 效果不明显 | 增加r值或训练数据量 |
| 显存不足 | 使用QLoRA或降低batch_size |
| 推理慢 | 合并权重后推理 |

### 目标模块选择

| 模块 | 效果 | 适用场景 |
|------|------|----------|
| q_proj + v_proj | ⭐⭐⭐ 基础 | 通用场景 |
| + k_proj + o_proj | ⭐⭐⭐⭐ 增强 | 长文本生成 |
| + gate_proj, up_proj, down_proj | ⭐⭐⭐⭐⭐ 全面 | 复杂任务 |

## 数据构建指南

### 指令数据格式
```json
{
  "instruction": "系统指令/任务描述",
  "input": "用户输入（可为空）",
  "output": "期望输出"
}
```

### 质量检查
- [ ] 数据格式正确
- [ ] 无隐私信息（脱敏）
- [ ] 领域知识准确
- [ ] 表达多样性
- [ ] 样本均衡（不同类型都有）

## 工具推荐

| 工具 | 用途 |
|------|------|
| LLaMA-Factory | 一站式微调框架 |
| Unsloth | 加速训练（2-3倍） |
| PEFT | LoRA/QLoRA实现 |
| Axolotl | 高级微调配置 |

## 适用场景

- ✅ 领域知识注入（医疗、法律、金融）
- ✅ 特定风格训练（正式/幽默）
- ✅ 指令遵循能力提升
- ✅ 私有数据问答

- ❌ 需要全量参数更新的任务
- ❌ 数据量超大（亿级）

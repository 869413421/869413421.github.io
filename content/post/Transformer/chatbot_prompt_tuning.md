---
title: "Transformer 学习之路 - Prompt Tuning 实战"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 Prompt Tuning 技术及其在 Transformer 模型中的应用"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - Prompt Tuning 实战

在深度学习的浩瀚宇宙中，Transformer 模型以其强大的表现力和广泛的应用场景，成为了自然语言处理（NLP）领域的明星。然而，随着模型规模的不断膨胀，如何高效地微调这些庞然大物以适应特定任务，成为了一个亟待解决的问题。今天，我们就来聊聊一种名为 **Prompt Tuning** 的参数高效微调方法，它如何在大型语言模型（如 GPT、BERT 等）中大显身手。

## Prompt Tuning 的基本原理

传统的微调方法通常需要调整整个模型的参数，这不仅计算资源消耗巨大，而且在大规模模型上实施起来颇为困难。Prompt Tuning 则另辟蹊径，它通过在输入前面加入一组可学习的提示（prompt）来引导模型产生期望的输出，而不是微调模型的全部参数。

### 关键点

1. **仅调整提示部分的参数**：Prompt Tuning 引入了一组可训练的嵌入向量，作为输入的提示词，将它们放置在输入文本的前面，以调整模型的输出行为。
2. **冻结模型权重**：模型的主干参数保持不变，只有提示词的嵌入参数在训练中被优化。

## Prompt Tuning 的优点

1. **减少计算资源和存储需求**：相比调整整个模型，Prompt Tuning 只更新提示词的嵌入向量，因而需要的计算和存储资源少得多。
2. **迁移性好**：在不同的任务上可以快速部署。尤其适合在大型模型上快速进行多任务适应。
3. **提升少样本学习能力**：Prompt Tuning 通过定制化提示引导模型，使得模型在少量样本任务中表现更为优异。

## Prompt Tuning 与微调的对比

| 特性               | 传统微调                | Prompt Tuning               |
|--------------------|-------------------------|-----------------------------|
| 调整的参数范围      | 整个模型参数            | 仅提示嵌入参数               |
| 计算资源需求        | 高                      | 低                          |
| 适用场景            | 大数据、复杂任务        | 少样本、特定任务            |
| 适用的模型大小      | 中小型模型             | 大型模型效果尤为显著         |

## 示例：Prompt Tuning 的应用场景

例如，如果我们要用 Prompt Tuning 来训练一个大型语言模型完成情感分类任务，可以采用以下步骤：

1. **定义提示向量**：给定一个输入句子“这个电影真棒！”，可以在前面加上一组初始的提示词，生成输入格式为：
   ```
   [提示词1] [提示词2] ... [提示词n] 这个电影真棒！
   ```
2. **训练提示词**：冻结模型参数，仅更新这些提示词，使得模型能够从中识别出情感信息。
3. **微调优化**：在情感分类数据集上优化这些提示词，以便在推理时引导模型准确地输出“正面”或“负面”情感分类。

## Soft Prompt 和 Hard Prompt 的对比

在 Prompt Tuning 中，**Soft Prompt** 和 **Hard Prompt** 是两种不同的提示方式，用于引导模型执行特定任务。

### Soft Prompt（软提示）

**Soft Prompt** 使用的是**可学习的嵌入向量**，而不是具体的文本词汇。其提示向量可以在训练过程中通过反向传播进行优化，从而让模型逐渐学会任务的上下文。

#### 特点

- **不需要具体的文本**：Soft Prompt 不需要语言学意义上的单词，而是直接插入到模型的输入嵌入空间中。
- **高度可适应性**：通过训练，Soft Prompt 向量会调整成对特定任务最有利的形式。
- **更适合大型语言模型**：在大型模型中，只微调提示嵌入向量，而不是整个模型参数，能显著降低计算资源需求。

### Hard Prompt（硬提示）

**Hard Prompt** 则是指具体的、由人类手动设计的文本提示。这类提示通常直接采用自然语言中的单词或短语，并将它们拼接到输入文本前或后，以此为模型提供上下文引导。

#### 特点

- **人为设定**：Hard Prompt 是显式的文本提示，可以为模型提供特定的上下文或结构。
- **灵活性较低**：一旦设计好，Hard Prompt 的内容就相对固定，难以适应更多任务。
- **更适合小型任务**：Hard Prompt 的设计可以帮助较小的模型理解任务，但对于更复杂的上下文和信息传递效果较差。

### 对比

| 特性            | Soft Prompt（软提示）              | Hard Prompt（硬提示）             |
|-----------------|-----------------------------------|-----------------------------------|
| 表达形式         | 嵌入向量（非语言）               | 明确的自然语言文本               |
| 是否需要训练     | 需要，提示向量会在训练中优化       | 不需要，提示文本人为设定          |
| 灵活性           | 高，适应性强                      | 较低，适用于特定任务             |
| 适用场景         | 大模型和多任务环境                | 小模型或特定领域的简单任务       |
| 实现难度         | 较高，需要训练调整                 | 较低，依赖经验和任务语境         |

## 代码实战

### 导入相关包

```python
from datasets import Dataset
from transformers import AutoTokenizer, AutoModelForCausalLM, DataCollatorForSeq2Seq, TrainingArguments, Trainer
```

### 加载数据集

```python
ds = Dataset.load_from_disk("/content/drive/MyDrive/ai-learning/3-PEFT/data/alpaca_data_zh/")
ds[:3]
```

### 数据集预处理

```python
tokenizer = AutoTokenizer.from_pretrained("Langboat/bloom-1b4-zh")

def process_func(example):
    MAX_LENGTH = 256
    input_ids, attention_mask, labels = [], [], []
    instruction = tokenizer("\n".join(["Human: " + example["instruction"], example["input"]]).strip() + "\n\nAssistant: ")
    response = tokenizer(example["output"] + tokenizer.eos_token)
    input_ids = instruction["input_ids"] + response["input_ids"]
    attention_mask = instruction["attention_mask"] + response["attention_mask"]
    labels = [-100] * len(instruction["input_ids"]) + response["input_ids"]
    if len(input_ids) > MAX_LENGTH:
        input_ids = input_ids[:MAX_LENGTH]
        attention_mask = attention_mask[:MAX_LENGTH]
        labels = labels[:MAX_LENGTH]
    return {
        "input_ids": input_ids,
        "attention_mask": attention_mask,
        "labels": labels
    }

tokenized_ds = ds.map(process_func, remove_columns=ds.column_names)
```

### 创建模型

```python
model = AutoModelForCausalLM.from_pretrained("Langboat/bloom-1b4-zh", low_cpu_mem_usage=True)
```

### 配置 Prompt Tuning

```python
from peft import PromptTuningConfig, get_peft_model, TaskType, PromptTuningInit

config = PromptTuningConfig(task_type=TaskType.CAUSAL_LM,
                            prompt_tuning_init=PromptTuningInit.TEXT,
                            prompt_tuning_init_text="下面是一段人与机器人的对话。",
                            num_virtual_tokens=len(tokenizer("下面是一段人与机器人的对话。")["input_ids"]),
                            tokenizer_name_or_path="Langboat/bloom-1b4-zh")

model = get_peft_model(model, config)
```

### 配置训练参数

```python
import os
os.environ["WANDB_DISABLED"] = "true"

args = TrainingArguments(
    output_dir="./chatbot",
    per_device_train_batch_size=1,
    gradient_accumulation_steps=8,
    logging_steps=10,
    num_train_epochs=1
)
```

### 创建训练器

```python
trainer = Trainer(
    model=model,
    args=args,
    tokenizer=tokenizer,
    train_dataset=tokenized_ds,
    data_collator=DataCollatorForSeq2Seq(tokenizer=tokenizer, padding=True),
)
```

### 模型训练

```python
trainer.train()
```

### 模型推理

```python
model = model.cuda()
ipt = tokenizer("Human: {}\n{}".format("考试有哪些技巧？", "").strip() + "\n\nAssistant: ", return_tensors="pt").to(model.device)
tokenizer.decode(model.generate(**ipt, max_length=128, do_sample=True)[0], skip_special_tokens=True)
```

## 总结

Prompt Tuning 作为一种轻量级的微调方法，不仅大幅降低了计算资源的需求，还在少样本学习和多任务适应方面展现了强大的潜力。通过本文的详细解析和代码实战，相信你已经对 Prompt Tuning 有了更深入的理解，并能够在实际项目中灵活运用。在未来的深度学习探索中，Prompt Tuning 无疑将成为你手中的一把利器，助你在 NLP 的海洋中乘风破浪。
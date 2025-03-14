---
title: "Transformer 学习之路 - 基于 T5 的文本摘要"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 Transformer 技术，特别是基于 T5 模型的文本摘要任务，涵盖技术原理、代码实现与应用场景。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 基于 T5 的文本摘要

文本摘要是一种自然语言处理（NLP）任务，旨在将长文本信息提炼为简洁的摘要，同时保留关键内容和语义。本文将深入探讨基于 T5 模型的文本摘要技术，涵盖其原理、实现步骤以及实际应用。

## 文本摘要的分类

文本摘要主要分为两类：

1. **抽取式摘要**：直接从原文中挑选出关键句子或短语，并组合成摘要。这种方法不会生成新内容，但效果依赖于原文的句子组织。

2. **生成式摘要**：使用生成算法基于原文创建新的句子和段落，可以进行词语重组和总结，适合灵活、内容丰富的摘要任务。生成式摘要往往能提供更自然、流畅的摘要。

## 序列到序列（Seq2Seq）模型

序列到序列（Seq2Seq）模型是解决生成式摘要任务的一种常用架构，适用于输入和输出都是序列的数据，例如机器翻译、文本摘要、对话生成等。

### Seq2Seq 基本结构

Seq2Seq模型一般由编码器（Encoder）和解码器（Decoder）组成：

- **编码器**：接收输入文本的序列，将其转换成隐含状态表示。

- **解码器**：根据编码器的输出和自身生成的上一步输出，逐步生成目标序列。

### Attention机制

在处理长文本摘要时，Seq2Seq模型的注意力机制（Attention）非常关键。Attention允许解码器在生成摘要的过程中动态关注输入文本中的重要部分，使摘要更精准，内容的连贯性更好。

### 常见的Seq2Seq模型

1. **RNN-based Seq2Seq**：早期的Seq2Seq模型多基于循环神经网络（RNN）或长短期记忆（LSTM）网络，但这些模型在长序列文本处理中性能不足。

2. **Transformer-based Seq2Seq**：目前最主流的是基于Transformer架构的Seq2Seq模型，如BERT、GPT等模型的变种。Transformer使用自注意力机制，可以在训练中更有效地捕获全局上下文，效果显著优于传统RNN模型。

## 经典文本摘要模型

1. **BERTSUM**：基于BERT的抽取式模型，设计了适合摘要任务的文本表示方式，适合抽取式摘要。

2. **T5（Text-to-Text Transfer Transformer）**：生成式Seq2Seq模型，能将各种文本处理任务（包括摘要）转换为统一的输入-输出格式。

3. **BART**：一种变换编码-解码的模型结构，可以针对文本摘要任务进行微调，擅长处理不规则的输入文本。

## 基于 T5 的文本摘要实现

### 安装依赖

```python
!pip install datasets rouge_chinese
```

### 导入相关包

```python
import torch
from datasets import Dataset
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM, DataCollatorForSeq2Seq, Seq2SeqTrainer, Seq2SeqTrainingArguments
```

### 加载数据集

```python
ds = Dataset.load_from_disk("/content/drive/MyDrive/ai-learning/2.NLP Task/08-text_summarization/nlpcc_2017/")
ds = ds.train_test_split(100, seed=42)
```

### 数据预处理

```python
tokenizer = AutoTokenizer.from_pretrained("Langboat/mengzi-t5-base")

def process_func(examples):
    contents = ["摘要生成: \n" + e for e in examples["content"]]
    inputs = tokenizer(contents, max_length=384, truncation=True)
    labels = tokenizer(text_target=examples["title"], max_length=64, truncation=True)
    inputs["labels"] = labels["input_ids"]
    return inputs

tokenized_ds = ds.map(process_func, batched=True)
```

### 创建模型

```python
model = AutoModelForSeq2SeqLM.from_pretrained("Langboat/mengzi-t5-base")
```

### 创建模型评估函数

```python
import numpy as np
from rouge_chinese import Rouge

rouge = Rouge()

def compute_metric(evalPred):
    predictions, labels = evalPred
    decode_preds = tokenizer.batch_decode(predictions, skip_special_tokens=True)
    labels = np.where(labels != -100, labels, tokenizer.pad_token_id)
    decode_labels = tokenizer.batch_decode(labels, skip_special_tokens=True)
    decode_preds = [" ".join(p) for p in decode_preds]
    decode_labels = [" ".join(l) for l in decode_labels]
    scores = rouge.get_scores(decode_preds, decode_labels, avg=True)
    return {
        "rouge-1": scores["rouge-1"]["f"],
        "rouge-2": scores["rouge-2"]["f"],
        "rouge-l": scores["rouge-l"]["f"],
    }
```

### 配置训练参数

```python
import os
os.environ["WANDB_DISABLED"] = "true"

args = Seq2SeqTrainingArguments(
    output_dir="./summary",
    per_device_train_batch_size=4,
    per_device_eval_batch_size=8,
    gradient_accumulation_steps=8,
    logging_steps=8,
    eval_strategy="epoch",
    save_strategy="epoch",
    metric_for_best_model="rouge-l",
    predict_with_generate=True
)
```

### 创建Trainer

```python
trainer = Seq2SeqTrainer(
    args=args,
    model=model,
    train_dataset=tokenized_ds["train"],
    eval_dataset=tokenized_ds["test"],
    compute_metrics=compute_metric,
    tokenizer=tokenizer,
    data_collator=DataCollatorForSeq2Seq(tokenizer=tokenizer)
)
```

### 训练模型

```python
trainer.train()
```

### 评估模型

```python
from transformers import pipeline

pipe = pipeline("text2text-generation", model=model, tokenizer=tokenizer, device=0)
pipe("摘要生成:\n" + ds["test"][-1]["content"], max_length=64, do_sample=True)
```

### 保存模型

```python
model.save_pretrained("/content/drive/MyDrive/ai-learning/2.NLP Task/08-text_summarization/summary")
```

## 总结

通过本文的讲解，我们深入了解了基于 T5 模型的文本摘要技术。从数据预处理到模型训练与评估，每一步都至关重要。希望本文能帮助你在实际项目中应用这些技术，生成高质量的自然语言摘要。
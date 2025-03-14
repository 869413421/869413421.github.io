---
title: "Transformer 学习之路 - Tokenizer 处理流程详解"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 Transformer 中的 Tokenizer 处理流程，包括分词、词典构建、数据转换、填充与截断等关键步骤，并结合代码示例进行详细讲解。"
categories: ["Python", "Transformer"]
---

# Tokenizer 处理流程详解

在 Transformer 模型中，Tokenizer（分词器）是将原始文本数据转换为模型可接受的输入格式的关键组件。本文将详细解析 Tokenizer 的处理流程，并结合代码示例帮助读者深入理解其工作原理。

## 1. 分词与词典构建

### Step1 分词
Tokenizer 的第一步是将文本数据进行分词。分词的方式可以是按字、按词或按子词（subword）进行。分词的目的是将连续的文本分割成离散的单元，以便后续处理。

```python
from transformers import AutoTokenizer

sen = "往者不可谏,来者犹可追"
tokenizer = AutoTokenizer.from_pretrained("uer/roberta-base-finetuned-dianping-chinese")
tokens = tokenizer.tokenize(sen)
print(tokens)
```

### Step2 构建词典
分词后，Tokenizer 会根据分词结果构建一个词典，将每个词或子词映射到一个唯一的 ID。词典的构建方式取决于是否使用预训练的词向量。如果使用预训练词向量，词典会根据词向量文件进行处理。

```python
# 查看词典详情
print(tokenizer.vocab)
print(tokenizer.vocab_size)
```

## 2. 数据转换与填充截断

### Step3 数据转换
Tokenizer 将分词后的文本序列转换为数字序列，因为神经网络只接受数字序列作为输入。

```python
# 将分词结果转换为 ID 序列
ids = tokenizer.convert_tokens_to_ids(tokens)
print(ids)

# 将 ID 序列转换回分词结果
tokens = tokenizer.convert_ids_to_tokens(ids)
print(tokens)

# 将分词序列转换为原始字符串
str_sen = tokenizer.convert_tokens_to_string(tokens)
print(str_sen)
```

### Step4 数据填充与截断
在以 batch 方式输入模型时，需要对过短的数据进行填充，对过长的数据进行截断，以确保所有数据的长度一致。

```python
# 填充，不足 15 个字会自动填充 0
ids = tokenizer.encode(sen, padding="max_length", max_length=15)
print(ids)

# 截断，超出 5 个字会被截断
ids = tokenizer.encode(sen, max_length=5, truncation=True)
print(ids)
```

## 3. 特殊标记与批量处理

### Step5 特殊标记
Tokenizer 会自动在句子前后添加特殊标记（如 `[CLS]` 和 `[SEP]`），以便模型识别句子的开始和结束。

```python
# 编码时添加特殊标记
ids = tokenizer.encode(sen, add_special_tokens=True)
print(ids)

# 解码时保留特殊标记
str_sen = tokenizer.decode(ids, skip_special_tokens=False)
print(str_sen)
```

### Step6 批量处理
Tokenizer 支持批量处理数据，只需传入一个句子列表即可。

```python
sens = ["弱小的我也有大梦想", "有梦想谁都了不起", "追逐梦想的心，比梦想本身，更可贵"]
res = tokenizer(sens)
print(res)
```

## 4. Fast/Slow Tokenizer

### Fast Tokenizer
Fast Tokenizer 基于 Rust 实现，速度更快，支持更多功能，如 `offset_mapping` 和 `word_id`。

```python
# 加载 Fast Tokenizer
fast_tokenizer = AutoTokenizer.from_pretrained("uer/roberta-base-finetuned-dianping-chinese")
inputs = fast_tokenizer(sen, return_offsets_mapping=True)
print(inputs)
```

### Slow Tokenizer
Slow Tokenizer 基于 Python 实现，速度较慢，功能相对较少。

```python
# 加载 Slow Tokenizer
slow_tokenizer = AutoTokenizer.from_pretrained("uer/roberta-base-finetuned-dianping-chinese", use_fast=False)
inputs = slow_tokenizer(sen)
print(inputs)
```

## 5. 远程加载与保存

### 远程加载 Tokenizer
Tokenizer 可以从 Hugging Face 远程加载，支持多种预训练模型。

```python
# 远程加载 Tokenizer
tokenizer = AutoTokenizer.from_pretrained("Skywork/Skywork-13B-base", trust_remote_code=True)
print(tokenizer)
```

### 保存与加载 Tokenizer
Tokenizer 可以保存到本地，以便后续使用。

```python
# 保存 Tokenizer 到本地
tokenizer.save_pretrained("skywork_tokenizer")

# 从本地加载 Tokenizer
tokenizer = AutoTokenizer.from_pretrained("skywork_tokenizer", trust_remote_code=True)
print(tokenizer)
```

## 总结

Tokenizer 是 Transformer 模型中不可或缺的一部分，它负责将原始文本转换为模型可接受的输入格式。通过本文的详细解析和代码示例，读者可以更好地理解 Tokenizer 的工作原理，并在实际项目中灵活应用。
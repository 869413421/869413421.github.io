---
title: "Transformer 学习之路 - 数据集加载与预处理"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析如何使用 datasets 库加载和预处理数据，为 Transformer 模型训练做好准备"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 数据集加载与预处理

在 Transformer 模型的训练过程中，数据集的加载和预处理是至关重要的一步。本文将详细介绍如何使用 `datasets` 库来加载、处理和管理数据集，确保数据能够高效地输入到模型中。

## 1. 安装与导入 `datasets` 库

首先，我们需要安装并导入 `datasets` 库。这个库提供了丰富的功能，可以轻松地加载和处理各种数据集。

```python
!pip install datasets
```

```python
from datasets import *
```

## 2. 加载数据集

`datasets` 库支持从线上和线下加载数据集。无论是公开的数据集还是自定义的数据集，都可以通过简单的代码实现加载。

### 2.1 加载线上数据集

```python
datasets = load_dataset("madao33/new-title-chinese")
datasets
```

### 2.2 加载特定任务的数据集

```python
boolq_dataset = load_dataset("super_glue", "boolq")
boolq_dataset
```

## 3. 数据集切分

在实际应用中，我们通常需要将数据集划分为训练集、验证集和测试集。`datasets` 库提供了多种切分方式，满足不同的需求。

### 3.1 加载部分数据

```python
# 只加载训练数据
dataset = load_dataset("madao33/new-title-chinese", split="train")
dataset
```

```python
# 只加载训练数据的10到100下标的数据
dataset = load_dataset("madao33/new-title-chinese", split="train[10:100]")
dataset
```

```python
# 取训练集中后50%数据
dataset = load_dataset("madao33/new-title-chinese", split="train[:50%]")
dataset
```

```python
# 先取后50%再取前50%
dataset = load_dataset("madao33/new-title-chinese", split=["train[:50%]", "train[50%:]"])
dataset
```

## 4. 查看数据集

在加载数据集后，我们通常需要查看数据的结构和内容，以便更好地理解数据。

### 4.1 查看数据集中的具体样本

```python
# 查看训练集第0个样本
datasets["train"][0]
```

```python
# 查看训练集前两个样本
datasets["train"][:2]
```

```python
# 获取前五个标题字段
datasets["train"]["title"][:5]
```

### 4.2 查看数据集的列信息

```python
# 查看列名
datasets["train"].column_names
```

```python
# 查看列详情
datasets["train"].features
```

## 5. 数据集划分

为了评估模型的性能，我们通常需要将数据集划分为训练集和测试集。

### 5.1 随机划分数据集

```python
# 划分数据集
dataset = datasets['train']
dataset.train_test_split(test_size=0.1)
```

### 5.2 按字段均衡划分数据集

```python
# 按数据中的字段来均衡划分
dataset = boolq_dataset["train"]
dataset.train_test_split(test_size=0.1, stratify_by_column="label")
```

## 6. 数据选取和过滤

在处理数据集时，我们可能需要根据特定条件选取或过滤数据。

### 6.1 选取数据

```python
# 选取
datasets["train"].select([0, 1])
```

### 6.2 过滤数据

```python
# 过滤
filter_dataset = datasets["train"].filter(lambda example: "中国" in example["title"])
filter_dataset["title"][:5]
```

## 7. 数据映射

数据映射是指对数据集中的每个样本进行某种操作，例如添加前缀或进行编码。

### 7.1 添加前缀

```python
def add_prefix(example):
    example["title"] = 'Prefix: ' + example["title"]
    return example

# 经过映射方法处理的数据集
prefix_dataset = datasets.map(add_prefix)
prefix_dataset["train"][:10]["title"]
```

### 7.2 使用 Tokenizer 进行编码

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-chinese")

def preprocess_function(example, tokenizer=tokenizer):
    model_inputs = tokenizer(example["content"], max_length=512, truncation=True)
    labels = tokenizer(example["title"], max_length=32, truncation=True)
    model_inputs["labels"] = labels["input_ids"]
    return model_inputs

processed_datasets = datasets.map(preprocess_function)
processed_datasets
```

### 7.3 多进程处理数据

```python
# 开启多进程处理数据，提高处理速度
processed_datasets = datasets.map(preprocess_function, num_proc=4)
processed_datasets
```

### 7.4 批量处理数据

```python
# 批量处理数据，去除多余的列
processed_datasets = datasets.map(preprocess_function, batched=True, remove_columns=datasets["train"].column_names)
processed_datasets
```

## 8. 保存和加载数据集

处理后的数据集可以保存到磁盘，以便后续使用。

### 8.1 保存数据集

```python
processed_datasets.save_to_disk("./processed_data")
```

### 8.2 加载数据集

```python
processed_datasets = load_from_disk("./processed_data")
processed_datasets
```

## 9. 加载本地文件作为数据集

`datasets` 库还支持加载本地文件作为数据集，例如 CSV、JSON 等格式。

### 9.1 加载 CSV 文件

```python
csv_path = "/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/ChnSentiCorp_htl_all.csv"
dataset = load_dataset("csv", data_files=csv_path, split="train")
dataset
```

### 9.2 加载文件夹内全部文件

```python
dataset = load_dataset("csv", data_files=["/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/all_data/ChnSentiCorp_htl_all copy 2.csv", "/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/all_data/ChnSentiCorp_htl_all copy 2.csv"], split='train')
dataset
```

## 10. 通过自定义脚本加载数据集

对于特殊格式的数据集，我们可以通过自定义脚本来加载和处理。

```python
dir_path = "/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/"
script_path = dir_path+"load_script.py"
dataset = load_dataset(script_path, split="train")
dataset
```

## 11. 使用 DataCollator 进行数据批处理

在训练模型时，我们通常需要将数据批处理为固定大小的张量。`DataCollator` 可以帮助我们实现这一目标。

### 11.1 使用 DataCollatorWithPadding

```python
from transformers import DataCollatorWithPadding

dataset = load_dataset("csv", data_files=csv_path, split='train')
dataset = dataset.filter(lambda x: x["review"] is not None)

def process_function(examples):
    tokenized_examples = tokenizer(examples["review"], max_length=128, truncation=True)
    tokenized_examples["labels"] = examples["label"]
    return tokenized_examples

tokenized_dataset = dataset.map(process_function, batched=True, remove_columns=dataset.column_names)
collator = DataCollatorWithPadding(tokenizer=tokenizer)

from torch.utils.data import DataLoader
dl = DataLoader(tokenized_dataset, batch_size=4, collate_fn=collator, shuffle=True)

num = 0
for batch in dl:
    print(batch["input_ids"].size())
    num += 1
    if num > 10:
        break
```

## 12. 总结

通过 `datasets` 库，我们可以轻松地加载、处理和
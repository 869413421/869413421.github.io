---
title: "Transformer 学习之路 - 数据处理与加载实战"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 Transformer 技术中的数据加载与处理，结合代码示例讲解如何使用 `datasets` 库高效处理 NLP 任务。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 数据处理与加载实战

在 Transformer 的学习过程中，数据处理是一个至关重要的环节。为了高效地加载和处理数据，我们通常会使用 `datasets` 库。本文将深入探讨如何使用 `datasets` 库进行数据加载、处理、映射和保存，并结合代码示例进行详细讲解。

## 1. `datasets` 库的基本使用

`datasets` 库是 Hugging Face 提供的一个强大工具，专门用于加载和处理各种数据集。它可以轻松地从线上或线下加载数据集，并提供了丰富的数据处理功能。

### 1.1 安装与导入

首先，我们需要安装 `datasets` 库：

```python
!pip install datasets
```

然后导入库：

```python
from datasets import *
```

### 1.2 加载线上数据集

使用 `load_dataset` 函数可以轻松加载线上数据集。例如，加载一个中文新闻标题数据集：

```python
datasets = load_dataset("madao33/new-title-chinese")
datasets
```

### 1.3 加载特定任务的数据集

有时我们只需要加载数据集中的某一项任务数据。例如，加载 `super_glue` 数据集中的 `boolq` 任务：

```python
boolq_dataset = load_dataset("super_glue", "boolq")
boolq_dataset
```

## 2. 数据集的切分与查看

在数据处理过程中，我们经常需要对数据集进行切分和查看。`datasets` 库提供了多种方式来实现这些操作。

### 2.1 切分数据集

我们可以通过 `split` 参数来切分数据集。例如，只加载训练数据：

```python
dataset = load_dataset("madao33/new-title-chinese", split="train")
dataset
```

还可以加载训练数据的特定部分，例如下标为 10 到 100 的数据：

```python
dataset = load_dataset("madao33/new-title-chinese", split="train[10:100]")
dataset
```

### 2.2 查看数据集

加载数据集后，我们可以查看数据集的具体内容。例如，查看训练集的第 0 个样本：

```python
datasets["train"][0]
```

或者查看训练集的前两个样本：

```python
datasets["train"][:2]
```

我们还可以获取特定字段的前五个值，例如标题字段：

```python
datasets["train"]["title"][:5]
```

## 3. 数据集的划分与过滤

在训练模型之前，我们通常需要将数据集划分为训练集和测试集，并对数据进行过滤。

### 3.1 划分数据集

使用 `train_test_split` 函数可以轻松划分数据集。例如，将数据集划分为训练集和测试集，测试集占 10%：

```python
dataset = datasets['train']
dataset.train_test_split(test_size=0.1)
```

### 3.2 数据过滤

我们可以根据特定条件对数据集进行过滤。例如，过滤出标题中包含“中国”的样本：

```python
filter_dataset = datasets["train"].filter(lambda example: "中国" in example["title"])
filter_dataset["title"][:5]
```

## 4. 数据映射与预处理

在将数据输入模型之前，通常需要对数据进行预处理。`datasets` 库提供了 `map` 函数，可以对数据集中的每个样本进行映射处理。

### 4.1 数据映射

我们可以定义一个映射函数，对数据集中的每个样本进行处理。例如，为每个标题添加前缀：

```python
def add_prefix(example):
    example["title"] = 'Prefix: ' + example["title"]
    return example

prefix_dataset = datasets.map(add_prefix)
prefix_dataset["train"][:10]["title"]
```

### 4.2 数据预处理

在 NLP 任务中，通常需要对文本进行分词处理。我们可以使用 `transformers` 库中的 `AutoTokenizer` 来进行分词。例如，使用 BERT 的分词器对文本进行编码：

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

为了提高处理速度，我们可以开启多进程处理：

```python
processed_datasets = datasets.map(preprocess_function, num_proc=4)
processed_datasets
```

## 5. 数据集的保存与加载

处理完数据集后，我们可以将其保存到磁盘，以便后续使用。

### 5.1 保存数据集

使用 `save_to_disk` 函数可以将数据集保存到磁盘：

```python
processed_datasets.save_to_disk("./processed_data")
```

### 5.2 加载数据集

使用 `load_from_disk` 函数可以从磁盘加载数据集：

```python
processed_datasets = load_from_disk("./processed_data")
processed_datasets
```

## 6. 从文件加载数据集

除了从线上加载数据集，我们还可以从本地文件加载数据集。

### 6.1 加载 CSV 文件

我们可以使用 `load_dataset` 函数加载本地 CSV 文件：

```python
csv_path = "/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/ChnSentiCorp_htl_all.csv"
dataset = load_dataset("csv", data_files=csv_path, split="train")
dataset
```

### 6.2 加载文件夹内的所有文件

我们还可以加载文件夹内的所有文件作为数据集：

```python
dataset = load_dataset("csv", data_files=["/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/all_data/ChnSentiCorp_htl_all copy 2.csv", "/content/drive/MyDrive/Colab Notebooks/transformers-code/01-Getting Started/05-datasets/all_data/ChnSentiCorp_htl_all copy 2.csv"], split='train')
dataset
```

## 7. 使用 DataCollator 进行数据批处理

在训练模型时，我们通常需要对数据进行批处理。`transformers` 库提供了 `DataCollatorWithPadding` 来处理填充问题。

### 7.1 数据批处理

首先，加载数据集并进行过滤：

```python
dataset = load_dataset("csv", data_files=csv_path, split='train')
dataset = dataset.filter(lambda x: x["review"] is not None)
dataset
```

然后，定义数据处理函数：

```python
def process_function(examples):
    tokenized_examples = tokenizer(examples["review"], max_length=128, truncation=True)
    tokenized_examples["labels"] = examples["label"]
    return tokenized_examples

tokenized_dataset = dataset.map(process_function, batched=True, remove_columns=dataset.column_names)
tokenized_dataset
```

最后，使用 `DataCollatorWithPadding` 进行批处理：

```python
collator = DataCollatorWithPadding(tokenizer=tokenizer)
```

### 7.2 创建 DataLoader

我们可以使用 `torch.utils.data.DataLoader` 来创建数据加载器：

```python
from torch.utils.data import DataLoader

dl = DataLoader(tokenized_dataset, batch_size=4, collate_fn=collator, shuffle=True)
```

查看批处理结果：

```python
num = 0
for batch in dl:
    print(batch["input_ids"].size())
    num += 1
    if num > 10:
        break
```

## 结语

通过本文的讲解，你应该已经掌握了如何使用 `datasets` 库进行数据加载、处理、映射和保存。这些技能在 Transformer 的学习和实践中至关重要，能够帮助你更高效地处理 NLP 任务。希望本文能为你提供有价值的参考，祝你在 Transformer 的学习之路上越走越远！
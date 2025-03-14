---
title: "Transformer 学习之路 - 文本分类实战"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析如何使用 Transformer 进行文本分类任务，涵盖数据集准备、模型训练与评估的全流程。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 文本分类实战

在这篇文章中，我们将通过一个完整的文本分类任务，深入解析 Transformer 技术的应用。我们将从数据集的准备开始，逐步完成模型的训练、评估和预测，并结合代码示例详细讲解每一步的实现细节。

## 1. 准备数据集

### 1.1 下载并保存数据集

首先，我们需要一个用于训练和测试的数据集。这里我们使用 `ChnSentiCorp_htl_all.csv` 数据集，它包含了酒店评论及其对应的情感标签（正面或负面）。

```python
import os
import requests

# 创建 'dataset' 文件夹（如果不存在）
os.makedirs("dataset", exist_ok=True)

# 文件 URL
url = "https://github.com/SophonPlus/ChineseNlpCorpus/raw/master/datasets/ChnSentiCorp_htl_all/ChnSentiCorp_htl_all.csv"

# 发出 GET 请求获取文件内容
response = requests.get(url)

# 文件保存路径
file_path = os.path.join("dataset", "ChnSentiCorp_htl_all.csv")

# 将内容保存到指定目录的文件中
with open(file_path, "wb") as file:
    file.write(response.content)
```

### 1.2 查看数据集

下载完成后，我们可以使用 `pandas` 来查看数据集的基本信息。

```python
import pandas as pd

# 读取保存在 'dataset' 文件夹中的 CSV 文件
df = pd.read_csv(file_path)

# 查看表头（列名）
print("表头（列名）：")
print(df.columns)

# 查看行数
print("\n行数：")
print(len(df))
```

### 1.3 去除空数据

为了确保数据质量，我们需要去除数据集中的空值。

```python
df = df.dropna()
df
```

## 2. 创建自定义数据集

接下来，我们使用 `datasets` 库来加载并处理数据集。

```python
!pip install datasets evaluate
```

```python
from transformers import DataCollatorWithPadding, AutoTokenizer
from datasets import load_dataset

csv_path = "dataset/ChnSentiCorp_htl_all.csv"
dataset = load_dataset("csv", data_files=csv_path, split='train')
dataset = dataset.filter(lambda x: x["review"] is not None)
dataset
```

## 3. 拆分数据集

为了进行模型训练和验证，我们需要将数据集拆分为训练集和验证集。

```python
datasets = dataset.train_test_split(test_size=0.1)
datasets
```

## 4. 数据预处理

在将数据输入模型之前，我们需要对文本进行预处理，包括分词和转换为模型可接受的格式。

```python
tokenizer = AutoTokenizer.from_pretrained("hfl/chinese-macbert-large")

def process_function(examples):
    # 减少 max_length，可以让训练数据减少，占用更少的内存
    tokenized_examples = tokenizer(examples["review"], max_length=128, truncation=True)
    tokenized_examples["labels"] = examples["label"]
    return tokenized_examples

tokenized_datasets = datasets.map(process_function, batched=True, remove_columns=datasets["train"].column_names)
tokenized_datasets
```

## 5. 创建模型及优化器

我们使用 `AutoModelForSequenceClassification` 来加载预训练模型，并准备进行微调。

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained("hfl/chinese-macbert-large")
```

## 6. 创建评估函数

为了评估模型的性能，我们定义了准确率和 F1 分数作为评估指标。

```python
import evaluate

acc_metric = evaluate.load("accuracy")
f1_metric = evaluate.load("f1")

def eval_metric(eval_predict):
    predictions, labels = eval_predict
    predictions = predictions.argmax(axis=-1)
    acc = acc_metric.compute(predictions=predictions, references=labels)
    f1 = f1_metric.compute(predictions=predictions, references=labels)
    acc.update(f1)
    return acc
```

## 7. 创建 TrainingArguments

`TrainingArguments` 是用于配置训练过程的参数集合。我们可以通过调整这些参数来优化训练效果。

```python
from transformers import TrainingArguments

train_args = TrainingArguments(
    output_dir="./checkpoints",      # 输出文件夹
    per_device_train_batch_size=64,   # 训练时的 batch_size
    optim="adafactor",               # 使用 Adafactor 优化器
    per_device_eval_batch_size=4,    # 验证时的 batch_size
    num_train_epochs=1,              # 训练轮数
    logging_steps=10,                # log 打印的频率
    eval_strategy="epoch",           # 每个 epoch 进行评估
    save_strategy="epoch",           # 每个 epoch 保存模型
    save_total_limit=3,              # 最大保存数
    learning_rate=2e-5,              # 学习率
    weight_decay=0.001,              # 权重衰减
    metric_for_best_model="f1",      # 使用 F1 作为评估指标
    load_best_model_at_end=True      # 训练完成后加载最优模型
)
```

## 8. 创建 Trainer

`Trainer` 是 Hugging Face 提供的一个高级 API，它封装了训练、评估和预测的过程。

```python
from transformers import DataCollatorWithPadding, Trainer

trainer = Trainer(
    model=model,
    args=train_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["test"],
    data_collator=DataCollatorWithPadding(tokenizer=tokenizer),
    compute_metrics=eval_metric
)
```

## 9. 训练模型

现在，我们可以开始训练模型了。

```python
for param in model.parameters():
    if not param.is_contiguous():
        param.data = param.contiguous()

trainer.train()
```

## 10. 模型评估

训练完成后，我们可以使用验证集来评估模型的性能。

```python
trainer.evaluate(tokenized_datasets["test"])
```

## 11. 模型预测

最后，我们可以使用训练好的模型对新的文本进行预测。

```python
from transformers import pipeline

id2_label = {0: "差评！", 1: "好评！"}
model.config.id2label = id2_label
pipe = pipeline("text-classification", model=model, tokenizer=tokenizer, device=0)

sen = "我觉得不错！"
pipe(sen)
```

## 总结

通过这个完整的文本分类实战，我们深入了解了如何使用 Transformer 进行自然语言处理任务。从数据集的准备到模型的训练、评估和预测，每一步都至关重要。希望这篇文章能帮助你更好地理解和应用 Transformer 技术。如果你有任何问题或建议，欢迎在评论区留言讨论！
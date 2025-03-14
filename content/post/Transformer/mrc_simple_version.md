```markdown
---
title: "Transformer 学习之路 - 机器阅读理解任务实现"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析如何使用 Transformer 实现机器阅读理解任务，涵盖数据预处理、模型训练与预测等关键步骤。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 机器阅读理解任务实现

在自然语言处理（NLP）领域，机器阅读理解（Machine Reading Comprehension, MRC）是一项核心任务，旨在让模型从给定的文本中提取出与问题相关的答案。Transformer 模型的出现极大地推动了这一领域的发展。本文将带你一步步实现一个基于 Transformer 的机器阅读理解任务。

## 1. 环境准备

首先，我们需要安装必要的库。这里我们使用 `datasets` 库来加载数据集，`transformers` 库来加载预训练模型和分词器。

```python
!pip install datasets
```

## 2. 导入相关包

接下来，我们导入所需的包，包括数据集加载、模型加载、分词器、训练参数配置等。

```python
from datasets import load_dataset
from transformers import AutoModelForQuestionAnswering, AutoTokenizer, TrainingArguments, Trainer, DefaultDataCollator
```

## 3. 加载数据集

我们使用 `cmrc2018` 数据集，这是一个中文机器阅读理解数据集。通过 `load_dataset` 函数加载数据集，并查看数据集的结构。

```python
datasets = load_dataset("cmrc2018", cache_dir="data")
datasets
```

查看训练集中的第一条数据：

```python
datasets["train"][0]
```

## 4. 数据预处理

数据预处理是机器阅读理解任务中非常关键的一步。我们需要将文本数据转换为模型可以接受的输入格式。这里我们使用 `AutoTokenizer` 加载预训练的分词器，并对数据进行编码。

```python
tokenizer = AutoTokenizer.from_pretrained("hfl/chinese-macbert-base")
tokenizer
```

为了便于理解，我们从训练集中随机选择 10 条数据进行处理。

```python
sample_dataset = datasets["train"].select(range(10))
sample_dataset[0]
```

对数据进行编码，返回的 `tokenized_examples` 是一个字典，包含 `input_ids`、`attention_mask`、`token_type_ids` 等。

```python
tokenized_examples = tokenizer(
    text=sample_dataset["context"],
    text_pair=sample_dataset["question"],
    return_offsets_mapping=True,
    truncation=True,
    max_length=512,
    padding="max_length",
    truncation_strategy="only_second",
)
tokenized_examples.keys()
```

## 5. 答案位置定位

在机器阅读理解任务中，我们需要找到答案在文本中的起始和结束位置。这里我们通过 `offset_mapping` 来定位答案在 token 中的位置。

```python
offset_mapping = tokenized_examples.pop("offset_mapping")

for idx, offset in enumerate(offset_mapping):
    answer = sample_dataset[idx]["answers"]
    start_char = answer["answer_start"][0]
    end_char = start_char + len(answer["text"][0])

    # 定位答案在token中的起始位置和结束位置
    context_start = tokenized_examples.sequence_ids(idx).index(1)
    context_end = tokenized_examples.sequence_ids(idx).index(None, context_start) - 1

    # 判断答案是否在context中
    if offset[context_end][1] < start_char or offset[context_start][0] > end_char:
        start_token_pos = 0
        end_token_pos = 0
    else:
        token_id = context_start
        while token_id <= context_end and offset[token_id][0] < start_char:
            token_id += 1
        start_token_pos = token_id

        token_id = context_end
        while token_id >= context_start and offset[token_id][1] > end_char:
            token_id -= 1
        end_token_pos = token_id

    print(answer, start_char, end_char, context_start, context_end, start_token_pos, end_token_pos)
    print("token answer decode:", tokenizer.decode(tokenized_examples["input_ids"][idx][start_token_pos: end_token_pos + 1]))
```

## 6. 数据处理函数

为了方便后续的训练，我们将上述过程封装成一个数据处理函数 `process_func`。

```python
def process_func(examples):
    tokenized_examples = tokenizer(
        text=examples["question"],
        text_pair=examples["context"],
        return_offsets_mapping=True,
        max_length=384,
        truncation="only_second",
        padding="max_length"
    )
    offset_mapping = tokenized_examples.pop("offset_mapping")
    start_positions = []
    end_positions = []

    for idx, offset in enumerate(offset_mapping):
        answer = examples["answers"][idx]
        start_char = answer["answer_start"][0]
        end_char = start_char + len(answer["text"][0])

        context_start = tokenized_examples.sequence_ids(idx).index(1)
        context_end = tokenized_examples.sequence_ids(idx).index(None, context_start) - 1

        if offset[context_end][1] < start_char or offset[context_start][0] > end_char:
            start_token_pos = 0
            end_token_pos = 0
        else:
            token_id = context_start
            while token_id <= context_end and offset[token_id][0] < start_char:
                token_id += 1
            start_token_pos = token_id

            token_id = context_end
            while token_id >= context_start and offset[token_id][1] > end_char:
                token_id -= 1
            end_token_pos = token_id

        start_positions.append(start_token_pos)
        end_positions.append(end_token_pos)

    tokenized_examples["start_positions"] = start_positions
    tokenized_examples["end_positions"] = end_positions
    return tokenized_examples
```

## 7. 数据集处理

使用 `map` 函数对整个数据集进行处理，并移除原始列。

```python
tokenized_datasets = datasets.map(process_func, batched=True, remove_columns=datasets["train"].column_names)
tokenized_datasets
```

## 8. 加载模型

接下来，我们加载预训练的问答模型 `AutoModelForQuestionAnswering`。

```python
model = AutoModelForQuestionAnswering.from_pretrained("hfl/chinese-macbert-base")
```

## 9. 配置训练参数

我们使用 `TrainingArguments` 来配置训练参数，包括输出目录、批量大小、评估策略等。

```python
args = TrainingArguments(
    output_dir="models_for_qa",
    per_device_train_batch_size=50,
    per_device_eval_batch_size=50,
    eval_strategy="epoch",
    save_strategy="epoch",
    logging_steps=50,
    num_train_epochs=3
)
```

## 10. 创建 Trainer

使用 `Trainer` 类来管理训练过程，包括模型、训练参数、数据集等。

```python
trainer = Trainer(
    model=model,
    args=args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["validation"],
    data_collator=DefaultDataCollator(),
    tokenizer=tokenizer
)
```

## 11. 训练模型

开始训练模型，确保模型参数是连续的。

```python
for param in model.parameters():
    if not param.is_contiguous():
        param.data = param.contiguous()
trainer.train()
```

## 12. 模型预测

训练完成后，我们可以使用模型进行预测。这里我们使用 `pipeline` 来简化预测过程。

```python
from transformers import pipeline

qa = pipeline("question-answering", model=model, tokenizer=tokenizer)
qa(question="信宜在哪里", context="广东茂名信宜的特色是三华李炒猪大肠。")
```

## 13. 模型保存

最后，我们将训练好的模型保存到 Google Drive 中，以便后续使用。

```python
from google.colab import drive
drive.mount('/content/drive')

model_save_path = "/content/drive/MyDrive/datasets/mrc"
model.save_pretrained(model_save_path)
```

## 总结

通过本文的步骤，我们实现了一个基于 Transformer 的机器阅读理解任务。从数据预处理到模型训练与预测，每一步都至关重要。希望这篇文章能帮助你更好地理解 Transformer 在机器阅读理解任务中的应用。
```
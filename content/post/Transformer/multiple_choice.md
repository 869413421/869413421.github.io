```markdown
---
title: "Transformer 学习之路 - 多项选择任务实现"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析如何使用 Transformer 技术实现多项选择任务，涵盖数据预处理、模型训练与评估等关键步骤。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 多项选择任务实现

在自然语言处理（NLP）领域，多项选择任务是一种常见的任务类型，旨在从给定的多个选项中选择最合适的答案。本文将深入探讨如何使用 Transformer 技术实现多项选择任务，并详细分析其中的技术原理和实现步骤。

## 1. 导入相关包

首先，我们需要导入必要的 Python 包。这些包包括 `evaluate`、`datasets` 和 `transformers`，它们分别用于评估模型性能、加载数据集以及构建和训练 Transformer 模型。

```python
!pip install evaluate

from google.colab import drive
import sys
drive.mount('/content/drive')

import evaluate
from datasets import load_dataset
from transformers import AutoModelForMultipleChoice, AutoTokenizer, TrainingArguments, Trainer
```

## 2. 加载数据集

接下来，我们加载 C3 数据集。C3 是一个中文多项选择数据集，包含上下文、问题和多个选项。我们首先加载数据集，并检查其结构和内容。

```python
c3 = load_dataset("clue/clue", "c3", cache_dir="./cache")
c3["train"][0]
c3['test'][0]

# 剔除 test 集，因为其 answer 为空
c3.pop("test")
c3
```

## 3. 数据预处理

在将数据输入模型之前，我们需要对其进行预处理。具体来说，我们将上下文和问题-选项对进行合并，并使用分词器将其转换为模型可接受的输入格式。

```python
tokenizer = AutoTokenizer.from_pretrained("hfl/chinese-macbert-base")

def process_function(examples):
    context = []
    question_choice = []
    labels = []

    for idx in range(len(examples["context"])):
        ctx = "\n".join([c.strip() for c in examples["context"][idx]]) if isinstance(examples["context"][idx], list) else examples["context"][idx].strip()
        question = examples["question"][idx]
        choices = examples["choice"][idx]

        for choice in choices:
            context.append(ctx)
            question_choice.append(question + " " + choice)

        if len(choices) < 4:
            for _ in range(4 - len(choices)):
                context.append(ctx)
                question_choice.append(question + " " + "不知道")

        labels.append(choices.index(examples["answer"][idx]))

    assert all(isinstance(c, str) for c in context), "Context 包含非字符串值"
    assert all(isinstance(qc, str) for qc in question_choice), "Question_choice 包含非字符串值"

    tokenized_examples = tokenizer(
        context,
        question_choice,
        truncation="only_first",
        max_length=256,
        padding="max_length",
        return_tensors="pt"
    )

    tokenized_examples = {k: [v[i: i + 4] for i in range(0, len(v), 4)] for k, v in tokenized_examples.items()}
    tokenized_examples["labels"] = labels

    return tokenized_examples

res = c3["train"].select(range(10)).map(process_function, batched=True)
res
```

## 4. 创建模型

我们使用 `AutoModelForMultipleChoice` 类来创建一个适用于多项选择任务的 Transformer 模型。这里我们选择了一个预训练的中文模型 `hfl/chinese-macbert-base`。

```python
model = AutoModelForMultipleChoice.from_pretrained("hfl/chinese-macbert-base")
```

## 5. 创建模型评估函数

为了评估模型的性能，我们定义了一个计算准确率的函数。该函数将模型的预测结果与真实标签进行比较，并返回准确率。

```python
import numpy as np

accuracy = evaluate.load("accuracy")

def compute_metric(pred):
    predictions, labels = pred
    predictions = np.argmax(predictions, axis=-1)
    return accuracy.compute(predictions=predictions, references=labels)
```

## 6. 配置训练参数

我们使用 `TrainingArguments` 类来配置模型的训练参数，包括批量大小、训练轮数、日志记录频率等。

```python
args = TrainingArguments(
    output_dir="./muliple_choice",
    per_device_train_batch_size=16,
    per_device_eval_batch_size=16,
    num_train_epochs=1,
    logging_steps=50,
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    fp16=True
)
```

## 7. 创建 Trainer

`Trainer` 类是 Hugging Face Transformers 库中的一个高级 API，用于简化模型的训练和评估过程。我们将模型、训练参数、数据集和评估函数传递给 `Trainer`。

```python
trainer = Trainer(
    model=model,
    args=args,
    tokenizer=tokenizer,
    train_dataset=tokenized_c3["train"],
    eval_dataset=tokenized_c3["validation"],
    compute_metrics=compute_metric
)
```

## 8. 训练模型

在配置好所有参数后，我们开始训练模型。训练过程中，模型会根据训练数据进行参数更新，并在每个 epoch 结束时进行评估。

```python
for param in model.parameters():
    if not param.is_contiguous():
        param.data = param.contiguous()
trainer.train()
```

## 9. 评估模型

训练完成后，我们使用自定义的流水线来评估模型的性能。该流水线将上下文、问题和选项作为输入，并返回模型预测的答案。

```python
from typing import Any
import torch

class MultipleChoicePipeline:
    def __init__(self, model, tokenizer) -> None:
        self.model = model
        self.tokenizer = tokenizer
        self.device = model.device

    def preprocess(self, context, quesiton, choices):
        cs, qcs = [], []
        for choice in choices:
            cs.append(context)
            qcs.append(quesiton + " " + choice)
        return tokenizer(cs, qcs, truncation="only_first", max_length=256, return_tensors="pt")

    def predict(self, inputs):
        inputs = {k: v.unsqueeze(0).to(self.device) for k, v in inputs.items()}
        return self.model(**inputs).logits

    def postprocess(self, logits, choices):
        predition = torch.argmax(logits, dim=-1).cpu().item()
        return choices[predition]

    def __call__(self, context, question, choices) -> Any:
        inputs = self.preprocess(context, question, choices)
        logits = self.predict(inputs)
        result = self.postprocess(logits, choices)
        return result

pipe = MultipleChoicePipeline(model, tokenizer)
pipe("我爱你你却爱着他", "你爱谁？", ["他", "你", "我"])
```

## 10. 保存模型

最后，我们将训练好的模型保存到指定路径，以便后续使用或部署。

```python
model_save_path = "/content/drive/MyDrive/ai-learning/2.NLP Task/04-multiple-choice/model"
model.save_pretrained(model_save_path)
```

通过以上步骤，我们成功地使用 Transformer 技术实现了一个多项选择任务。希望本文能帮助你更好地理解 Transformer 在 NLP 任务中的应用。
```
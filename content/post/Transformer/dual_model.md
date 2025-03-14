---
title: "Transformer 学习之路 - 文本相似度实战"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 Transformer 技术在文本相似度任务中的应用，结合代码示例详细讲解技术原理与实现。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 文本相似度实战

Transformer 模型在自然语言处理（NLP）领域取得了巨大的成功，尤其是在文本相似度任务中表现尤为突出。本文将结合代码示例，深入解析如何利用 Transformer 技术实现文本相似度计算，并详细分析其背后的技术原理。

## 1. 背景与问题

文本相似度任务是 NLP 中的一项基础任务，旨在衡量两段文本在语义上的相似程度。常见的应用场景包括问答系统、信息检索、文本匹配等。传统的文本相似度计算方法（如 TF-IDF、余弦相似度等）往往难以捕捉文本的深层语义信息，而 Transformer 模型通过自注意力机制（Self-Attention）能够更好地理解文本的上下文关系，从而提升相似度计算的准确性。

## 2. 技术原理

### 2.1 Transformer 模型

Transformer 模型的核心是自注意力机制，它能够动态地计算输入序列中每个词与其他词的相关性，从而捕捉长距离依赖关系。Transformer 模型由编码器（Encoder）和解码器（Decoder）两部分组成，但在文本相似度任务中，我们通常只使用编码器部分。

### 2.2 文本相似度计算

在文本相似度任务中，我们需要将两段文本分别输入到 Transformer 模型中，获取它们的向量表示，然后通过余弦相似度等方法来衡量它们的相似程度。为了实现这一目标，我们可以使用预训练的 Transformer 模型（如 BERT）作为基础模型，并在其基础上进行微调。

## 3. 代码实现

### 3.1 环境准备

首先，我们需要安装必要的库，并加载数据集。

```python
!pip install evaluate datasets
```

```python
import sys
from google.colab import drive

drive.mount('/content/drive')
sys.path.append('/content/drive/MyDrive/ai-learning/2.NLP Task/05-sentence_similarity')
```

### 3.2 加载数据集

我们使用 `datasets` 库加载一个包含句子对和标签的数据集。

```python
from datasets import load_dataset

dataset = load_dataset("json", data_files="/content/drive/MyDrive/ai-learning/2.NLP Task/05-sentence_similarity/train_pair_1w.json", split="train")
datasets = dataset.train_test_split(test_size=0.2)
```

### 3.3 数据预处理

使用预训练的 `hfl/chinese-macbert-base` 分词器对文本进行预处理。

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("hfl/chinese-macbert-base")

def preprocess_function(examples):
    sentences = []
    labels = []
    for sentence1, sentence2, label in zip(examples["sentence1"], examples["sentence2"], examples["label"]):
        sentences.append(sentence1)
        sentences.append(sentence2)
        labels.append(1 if label == 1 else -1)

    tokenized_examples = tokenizer(
        sentences,
        max_length=128,
        padding="max_length",
        truncation=True
    )

    tokenized_examples = {k: [v[i:i + 2] for i in range(0, len(v), 2)] for k, v in tokenized_examples.items()}
    tokenized_examples["labels"] = labels
    return tokenized_examples

tokenized_datasets = datasets.map(preprocess_function, batched=True, remove_columns=datasets["train"].column_names)
```

### 3.4 创建模型

我们定义了一个 `DualModel` 类，用于处理句子对的相似度计算。

```python
import torch
from transformers import BertPreTrainedModel, BertModel
from torch.nn import CosineSimilarity, CosineEmbeddingLoss

class DualModel(BertPreTrainedModel):
    def __init__(self, config: PretrainedConfig, *inputs, **kwargs):
        super().__init__(config, *inputs, **kwargs)
        self.bert = BertModel(config)
        self.post_init()

    def forward(self, input_ids, attention_mask, token_type_ids, labels=None):
        senA_input_ids, senB_input_ids = input_ids[:, 0], input_ids[:, 1]
        senA_attention_mask, senB_attention_mask = attention_mask[:, 0], attention_mask[:, 1]
        senA_token_type_ids, senB_token_type_ids = token_type_ids[:, 0], token_type_ids[:, 1]

        senA_outputs = self.bert(senA_input_ids, attention_mask=senA_attention_mask, token_type_ids=senA_token_type_ids)
        senA_pooled_output = senA_outputs[1]

        senB_outputs = self.bert(senB_input_ids, attention_mask=senB_attention_mask, token_type_ids=senB_token_type_ids)
        senB_pooled_output = senB_outputs[1]

        cos = CosineSimilarity()(senA_pooled_output, senB_pooled_output)

        loss = None
        if labels is not None:
            loss_fct = CosineEmbeddingLoss(0.3)
            loss = loss_fct(senA_pooled_output, senB_pooled_output, labels)

        output = (cos,)
        return ((loss,) + output) if loss is not None else output

model = DualModel.from_pretrained("hfl/chinese-macbert-base")
```

### 3.5 模型训练与评估

我们使用 `Trainer` 类来训练模型，并定义评估函数来计算准确率和 F1 分数。

```python
from transformers import Trainer, TrainingArguments
import evaluate

acc_metric = evaluate.load("accuracy")
f1_metric = evaluate.load("f1")

def eval_metric(eval_predict):
    predictions, labels = eval_predict
    predictions = [int(p > 0.7) for p in predictions]
    labels = [int(l > 0) for l in labels]
    acc = acc_metric.compute(predictions=predictions, references=labels)
    f1 = f1_metric.compute(predictions=predictions, references=labels)
    acc.update(f1)
    return acc

train_args = TrainingArguments(
    output_dir="./dual_model",
    per_device_train_batch_size=32,
    per_device_eval_batch_size=32,
    logging_steps=10,
    eval_strategy="epoch",
    save_strategy="epoch",
    save_total_limit=3,
    learning_rate=2e-5,
    weight_decay=0.01,
    metric_for_best_model="f1",
    load_best_model_at_end=True
)

trainer = Trainer(
    model=model,
    args=train_args,
    tokenizer=tokenizer,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["test"],
    compute_metrics=eval_metric
)

trainer.train()
trainer.evaluate(tokenized_datasets["test"])
```

### 3.6 模型预测与保存

最后，我们定义了一个 `SentenceSimilarityPipeline` 类来进行模型预测，并将模型保存到指定路径。

```python
class SentenceSimilarityPipeline:
    def __init__(self, model, tokenizer) -> None:
        self.model = model.bert
        self.tokenizer = tokenizer
        self.device = model.device

    def preprocess(self, senA, senB):
        return self.tokenizer([senA, senB], max_length=128, truncation=True, return_tensors="pt", padding=True)

    def predict(self, inputs):
        inputs = {k: v.to(self.device) for k, v in inputs.items()}
        return self.model(**inputs)[1]

    def postprocess(self, logits):
        cos = CosineSimilarity()(logits[None, 0, :], logits[None,1, :]).squeeze().cpu().item()
        return cos

    def __call__(self, senA, senB, return_vector=False):
        inputs = self.preprocess(senA, senB)
        logits = self.predict(inputs)
        result = self.postprocess(logits)
        if return_vector:
            return result, logits
        else:
            return result

pipe = SentenceSimilarityPipeline(model, tokenizer)
print(pipe("我喜欢北京", "北京是我喜欢的城市", return_vector=True))

model_save_path = "/content/drive/MyDrive/ai-learning/2.NLP Task/05-sentence_similarity/model/dual"
model.save_pretrained(model_save_path)
```

## 4. 总结

本文详细介绍了如何利用 Transformer 技术实现文本相似度计算，并通过代码示例展示了从数据预处理、模型训练到预测的完整流程。通过使用预训练的 Transformer 模型，我们能够更好地捕捉文本的语义信息，从而提升文本相似度任务的准确性。希望本文能帮助读者深入理解 Transformer 技术，并在实际项目中应用这些技术。
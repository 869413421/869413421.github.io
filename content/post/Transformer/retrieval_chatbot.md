---
title: "Transformer 学习之路 - 检索机器人实战"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析如何利用 Transformer 技术构建检索机器人，涵盖数据读取、模型加载、向量编码、索引创建与匹配等关键步骤。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - 检索机器人实战

在自然语言处理（NLP）领域，Transformer 模型已经成为许多任务的核心工具。今天，我们将通过一个实际案例——构建一个**检索机器人**，来深入探讨 Transformer 的应用。这个机器人能够根据用户的问题，从 FAQ 数据集中检索出最相关的答案。

## 1. 读取 FAQ 数据

首先，我们需要准备好 FAQ 数据集。这里我们使用了一个法律相关的 FAQ 数据集 `law_faq.csv`。

```python
import pandas as pd

data = pd.read_csv("/content/drive/MyDrive/ai-learning/2.NLP Task/06-retrieval_chatbot/law_faq.csv")
data.head()
```

### 为什么需要这一步？
FAQ 数据集是我们机器人的“知识库”，它包含了常见问题及其对应的答案。读取数据后，我们才能进行后续的处理和检索。

## 2. 加载预训练模型

接下来，我们需要加载预训练的 Transformer 模型。这里我们使用了 `DualModel` 和 `AutoTokenizer`。

```python
from dual_model import DualModel
from transformers import AutoTokenizer

dual_model = DualModel.from_pretrained(dual_model_path)
dual_model.cuda()
dual_model.eval()

tokenizer = AutoTokenizer.from_pretrained("hfl/chinese-macbert-base")
```

### 为什么需要这一步？
预训练模型已经在大规模数据上进行了训练，能够捕捉到丰富的语言特征。加载这些模型后，我们可以直接使用它们来对文本进行编码，而不需要从头开始训练。

## 3. 将问题编码转换为向量

为了进行检索，我们需要将 FAQ 中的问题转换为向量表示。这里我们使用了 `DualModel` 的 BERT 模型来生成向量。

```python
import torch
from tqdm import tqdm

questions = data["title"].to_list()
vectors = []

with torch.inference_mode():
    for i in tqdm(range(0, len(questions), 32)):
        batch_sens = questions[i: i + 32]
        inputs = tokenizer(batch_sens, return_tensors="pt", padding=True, max_length=128, truncation=True)
        inputs = {k: v.to(dual_model.device) for k, v in inputs.items()}
        vector = dual_model.bert(**inputs)[1]
        vectors.append(vector)

vectors = torch.concat(vectors, dim=0).cpu().numpy()
vectors.shape
```

### 为什么需要这一步？
将文本转换为向量后，我们可以通过计算向量之间的相似度来找到最相关的问题。这种向量化的表示方式使得检索过程更加高效和准确。

## 4. 创建索引，存储向量

为了快速检索，我们需要将生成的向量存储在索引中。这里我们使用了 `faiss` 库来创建和存储索引。

```python
import faiss

index = faiss.IndexFlatIP(768)
faiss.normalize_L2(vectors)
index.add(vectors)
```

### 为什么需要这一步？
索引能够加速向量检索的过程。通过将向量存储在索引中，我们可以在毫秒级别内找到与用户问题最相关的 FAQ。

## 5. 对用户提问进行编码

当用户提出问题时，我们同样需要将其转换为向量。

```python
question = "寻衅滋事"

with torch.inference_mode():
    inputs = tokenizer(question, return_tensors="pt", padding=True, max_length=128, truncation=True)
    inputs = {k: v.to(dual_model.device) for k, v in inputs.items()}
    vector = dual_model.bert(**inputs)[1]
    q_vector = vector.cpu().numpy()
```

### 为什么需要这一步？
用户问题的向量表示是检索的基础。只有将问题转换为向量，我们才能通过索引找到最相关的问题。

## 6. 向量匹配（根据向量召回）

接下来，我们通过 `faiss` 索引来找到与用户问题最相关的 FAQ。

```python
faiss.normalize_L2(q_vector)
scores, indexes = index.search(q_vector, 10)
topk_result = data.values[indexes[0].tolist()]
topk_result[:, 0]
```

### 为什么需要这一步？
向量匹配是检索的核心步骤。通过计算用户问题向量与 FAQ 向量之间的相似度，我们可以找到最相关的问题。

## 7. 加载交互模型，用于匹配最相似的 FAQ

为了进一步提高检索的准确性，我们加载了一个交互模型 `AutoModelForSequenceClassification`。

```python
from transformers import AutoModelForSequenceClassification

corss_model = AutoModelForSequenceClassification.from_pretrained(cross_model_path, num_labels=1)
corss_model = corss_model.cuda()
corss_model.eval()
```

### 为什么需要这一步？
交互模型能够更精确地计算问题与候选答案之间的匹配度。通过这个模型，我们可以从候选答案中选择最合适的答案。

## 8. 最终预测

最后，我们使用交互模型对候选答案进行打分，并选择最合适的答案。

```python
canidate = topk_result[:, 0].tolist()
ques = [question] * len(canidate)
inputs = tokenizer(ques, canidate, return_tensors="pt", padding=True, max_length=128, truncation=True)
inputs = {k: v.to(corss_model.device) for k, v in inputs.items()}

with torch.inference_mode():
    logits = corss_model(**inputs).logits.squeeze()
    result = torch.argmax(logits, dim=-1)

canidate_answer = topk_result[:, 1].tolist()
match_quesiton = canidate[result.item()]
final_answer = canidate_answer[result.item()]
match_quesiton, final_answer
```

### 为什么需要这一步？
最终预测是检索机器人的最后一步。通过交互模型的选择，我们能够确保返回给用户的答案是最准确和相关的。

## 总结

通过以上步骤，我们成功构建了一个基于 Transformer 的检索机器人。这个机器人能够从 FAQ 数据集中快速找到与用户问题最相关的答案。整个过程涉及了数据读取、模型加载、向量编码、索引创建与匹配等多个关键步骤，展示了 Transformer 在 NLP 任务中的强大能力。

希望这篇文章能够帮助你更好地理解 Transformer 的应用，并为你的 NLP 项目提供一些灵感。如果你有任何问题或建议，欢迎在评论区留言讨论！
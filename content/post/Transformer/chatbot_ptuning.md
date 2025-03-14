```markdown
---
title: "Transformer 学习之路 - P-tuning 技术深入解析"
date: 2024-04-19T17:35:18+08:00
draft: false
description: "深入解析 P-tuning 技术，探讨其核心思想、工作原理、优点及应用场景，并结合代码示例进行讲解。"
categories: ["Python", "Transformer"]
---

# Transformer 学习之路 - P-tuning 技术深入解析

P-tuning 是 Prompt Tuning 的一种增强版本，最早由清华大学提出，设计初衷是通过优化提示词来提升预训练模型在下游任务上的表现。P-tuning 不仅仅是简单的 Prompt Tuning，而是进一步利用**可学习的嵌入向量**来模拟提示词，以适应特定任务需求。P-tuning 特别适合语言模型微调，不仅适用于小规模的下游任务，也适用于需要更高灵活性的大模型。

## P-tuning 的核心思想

与普通的 Prompt Tuning 通过固定的提示词嵌入来引导模型不同，P-tuning 使用一组可训练的嵌入向量作为提示词，这些嵌入向量会在训练过程中自动调整，生成特定任务的嵌入表示。相比普通的 Prompt Tuning，P-tuning 的灵活性和适应性更高。

## P-tuning 的工作原理

1. **初始化嵌入向量**：
   - 与普通 Prompt Tuning 类似，P-tuning 初始化时也会创建一组提示词嵌入向量（即提示 token），但这些向量是随机初始化的，没有特定的含义。

2. **提示词和输入拼接**：
   - 将这些嵌入向量与实际输入文本拼接，形成输入序列。例如，对于句子 "I love this product!"，模型接收的输入会是：
     ```
     [提示1] [提示2] ... [提示N] I love this product!
     ```

3. **训练过程**：
   - 在训练过程中，提示嵌入会自动调整以适应任务需求。模型通过反向传播更新提示词的嵌入向量，以便它们能有效引导模型生成或理解特定任务的输出。

4. **动态适应性**：
   - P-tuning 提供了动态适应能力。提示词嵌入的值会随着任务的不同而变化，从而让模型更加灵活地适应各类任务。

5. **冻结模型参数**：
   - P-tuning 通常会冻结模型的其他参数，仅更新提示词嵌入。这种方法显著减少了计算需求，使其能够更高效地在大规模预训练模型上微调。

## P-tuning 的优点

- **高效性**：只优化少量的提示词嵌入，减少了训练时的计算开销。
- **灵活性**：适合适应多种下游任务，对输入内容和任务需求的变化具有较强适应性。
- **适合多种模型**：P-tuning 能有效应用在大规模预训练语言模型上，尤其在使用如 GPT、BERT 之类的模型时效果显著。

## P-tuning 的应用场景

P-tuning 通常用于以下场景：

- **文本分类**：例如情感分析、意图检测等任务。
- **生成式任务**：例如文本生成、摘要生成等任务。
- **信息抽取**：如命名实体识别、关系抽取等任务。

## 举个例子

假设有一个情感分类任务，输入是 “This movie was fantastic!”。通过 P-tuning，可以将输入的内容和可学习的提示词嵌入向量拼接在一起：

```plaintext
[Prompt1] [Prompt2] ... [PromptN] This movie was fantastic!
```

在训练过程中，模型会自动调整提示词的嵌入向量，使其能够引导模型更准确地预测情感类别。

## 总结

P-tuning 是 Prompt Tuning 的一种改进方法，利用了可训练的嵌入向量作为提示，使得模型可以更有效地适应特定的下游任务。这种方法能够提高模型的灵活性，同时降低训练成本，非常适合大规模预训练模型在多任务上的高效微调。

## 代码示例

### Step1 导入相关包

```python
from datasets import Dataset
from transformers import AutoTokenizer, AutoModelForCausalLM, DataCollatorForSeq2Seq, TrainingArguments, Trainer
```

### Step2 加载数据集

```python
ds = Dataset.load_from_disk("/content/drive/MyDrive/ai-learning/3-PEFT/data/alpaca_data_zh/")
ds[:3]
```

### Step3 数据集预处理

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
tokenizer.decode(tokenized_ds[1]["input_ids"])
tokenizer.decode(list(filter(lambda x: x != -100, tokenized_ds[1]["labels"])))
```

### Step4 创建模型

```python
model = AutoModelForCausalLM.from_pretrained("Langboat/bloom-1b4-zh")
```

### P-tuning

#### PEFT Step1 配置文件

```python
from peft import PromptEncoderConfig, TaskType, get_peft_model, PromptEncoderReparameterizationType

config = PromptEncoderConfig(task_type=TaskType.CAUSAL_LM, num_virtual_tokens=10,
                             encoder_reparameterization_type=PromptEncoderReparameterizationType.MLP,
                             encoder_dropout=0.1, encoder_num_layers=5, encoder_hidden_size=1024)
config
```

#### PEFT Step2 创建模型

```python
model = get_peft_model(model, config)
model.print_trainable_parameters()
```

### Step5 配置训练参数

```python
args = TrainingArguments(
    output_dir="./chatbot",
    per_device_train_batch_size=1,
    gradient_accumulation_steps=8,
    logging_steps=10,
    num_train_epochs=1
)
```

### Step6 创建训练器

```python
trainer = Trainer(
    model=model,
    args=args,
    tokenizer=tokenizer,
    train_dataset=tokenized_ds,
    data_collator=DataCollatorForSeq2Seq(tokenizer=tokenizer, padding=True),
)
```

### Step7 模型训练

```python
trainer.train()
```

### Step8 模型推理

```python
model = model.cuda()
ipt = tokenizer("Human: {}\n{}".format("数学考试有哪些技巧？", "").strip() + "\n\nAssistant: ", return_tensors="pt").to(model.device)
print(tokenizer.decode(model.generate(**ipt, max_length=256, do_sample=True)[0], skip_special_tokens=True))
```

通过以上步骤，我们深入了解了 P-tuning 技术的核心思想、工作原理及其在实际应用中的优势。希望这篇文章能帮助你在 Transformer 的学习之路上更进一步！
```
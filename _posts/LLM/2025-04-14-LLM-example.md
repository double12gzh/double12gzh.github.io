---
layout: post
title: 大模型推理示例
categories: [LLM]
description: 用示例说明大模型推理背后的具体步骤
keywords: LLM, 推理, tokenization, transformer
excerpt: 摘要：用示例说明大模型推理背后的具体步骤
---

**目录**

* TOC
{:toc}

## 1. 背景

在深度学习领域，大模型（如 GPT-3、BERT 等）的推理是一个复杂且资源密集型的过程。这些模型的训练通常涉及大量的参数和计算资源，而推理则是在给定输入数据(即 prompt)的情况下生成输出或预测结果(completion)。

## 2. 大模型推理的基本步骤

### 步骤1：Tokenization

**描述**: 将原始文本转换为模型可以理解的格式。这涉及到将单词拆分成更小的单元（通常是子词单位），以便于模型处理。

这部分操作是在 CPU 上完成的，因为 tokenization 通常不涉及大量的计算。输入是 prompt，输出是 token ids，输出的结果最终会被送到 GPU/NPU 显存中已便进行下一步的计算。

**示例代码 (Python 使用 Hugging Face 的 Transformers 库)**:

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
inputs = tokenizer("Hello, world!", return_tensors="pt")
print(inputs)
```

### 步骤2：加载预训练的大模型

**描述**: 从磁盘或其他存储位置加载已经训练好的模型权重。

**示例代码 (Python 使用 Hugging Face 的 Transformers 库)**:

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained('bert-base-uncased')
print(model)
```

### 步骤3：模型推理

**描述**: 使用模型对输入数据进行推理，并获取输出结果。
**示例代码 (Python 使用 Hugging Face 的 Transformers 库)**:

```python
from transformers import pipeline
classifier = pipeline("sentiment-analysis")
result = classifier("I love Hugging Face")
print(result)
```

> 详细的推理过程可以参考 [大模型推理加速与KV Cache（一）：什么是KV Cache](https://zhuanlan.zhihu.com/p/717581669)

## 3. 详细代码示例

使用 GPT2 模型，对输入（prompt）`What color is the sky?` 进行推理，并将生成的结果（completion）、token ids 打印出来。

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM


def print_decode_result(tokenizer: AutoTokenizer, outputs: torch.Tensor):
    # 解码并打印结果（跳过特殊符号）
    print(tokenizer.decode(outputs[0], skip_special_tokens=True))


def print_output_tensor(outputs: torch.Tensor):
    # 直接打印生成的Token张量（不解码）
    print("生成的Token张量: \n", outputs)


def print_token_details(tokenizer: AutoTokenizer, outputs: torch.Tensor):
    # 提取生成的 Token ID 列表（包括输入和生成部分）
    token_ids = outputs[0].tolist()  # 形状为 [sequence_length]

    # 将 Token ID 转换为对应的分词文本
    tokens = tokenizer.convert_ids_to_tokens(token_ids)  # 直接处理整个序列

    # 对齐显示 Token ID 和分词
    print("Token ID  |  Token Text")
    print("------------------------")
    for token_id, token_text in zip(token_ids, tokens):
        print(f"{token_id:8}  |  {token_text}")


def generate_text(
    prompt: str,
    model_name: str = "gpt2",
    max_length: int = 50,
    temperature: float = 0.7,
    do_sample: bool = False,
) -> tuple[AutoTokenizer, torch.Tensor]:
    # 加载模型和分词器
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForCausalLM.from_pretrained(model_name)

    # 编码输入文本
    inputs = tokenizer(prompt, return_tensors="pt")

    # 生成输出（添加生成参数）
    outputs = model.generate(
        **inputs,
        max_length=max_length,
        pad_token_id=tokenizer.eos_token_id,  # 使用 GPT-2 的结束符作为 pad token
        eos_token_id=tokenizer.eos_token_id,
        do_sample=do_sample,
        temperature=temperature,
    )
    # 返回分词器、输入文本和生成的Token张量
    return tokenizer, inputs, outputs


# 生成文本并打印结果
tokenizer, inputs, outputs = generate_text("What color is the sky?")

# 1. 解码并打印结果（跳过特殊符号）
print_decode_result(tokenizer, outputs)

# 2. 直接打印生成的Token张量（不解码）
print_output_tensor(outputs)

# 3. 打印Token详细信息
print_token_details(tokenizer, outputs)

# 4. 打印输入文本
print("输入文本: \n", inputs)
```

上述代码的执行结果如下：

```bash
What color is the sky?

The sky is the most beautiful thing in the world. It's the most beautiful thing in the world because it's the only thing that can change the world. It's the only thing that can change the world
生成的Token张量: 
 tensor([[2061, 3124,  318,  262, 6766,   30,  198,  198,  464, 6766,  318,  262,
          749, 4950, 1517,  287,  262,  995,   13,  632,  338,  262,  749, 4950,
         1517,  287,  262,  995,  780,  340,  338,  262,  691, 1517,  326,  460,
         1487,  262,  995,   13,  632,  338,  262,  691, 1517,  326,  460, 1487,
          262,  995]])
Token ID  |  Token Text
------------------------
    2061  |  What
    3124  |  Ġcolor
     318  |  Ġis
     262  |  Ġthe
    6766  |  Ġsky
      30  |  ?
     198  |  Ċ
     198  |  Ċ
     464  |  The
    6766  |  Ġsky
     318  |  Ġis
     262  |  Ġthe
     749  |  Ġmost
    4950  |  Ġbeautiful
    1517  |  Ġthing
     287  |  Ġin
     262  |  Ġthe
     995  |  Ġworld
      13  |  .
     632  |  ĠIt
     338  |  's
     262  |  Ġthe
     749  |  Ġmost
    4950  |  Ġbeautiful
    1517  |  Ġthing
     287  |  Ġin
     262  |  Ġthe
     995  |  Ġworld
     780  |  Ġbecause
     340  |  Ġit
     338  |  's
     262  |  Ġthe
     691  |  Ġonly
    1517  |  Ġthing
     326  |  Ġthat
     460  |  Ġcan
    1487  |  Ġchange
     262  |  Ġthe
     995  |  Ġworld
      13  |  .
     632  |  ĠIt
     338  |  's
     262  |  Ġthe
     691  |  Ġonly
    1517  |  Ġthing
     326  |  Ġthat
     460  |  Ġcan
    1487  |  Ġchange
     262  |  Ġthe
     995  |  Ġworld
输入文本: 
 {'input_ids': tensor([[2061, 3124,  318,  262, 6766,   30]]), 'attention_mask': tensor([[1, 1, 1, 1, 1, 1]])}
```

> [colab 地址](https://colab.research.google.com/drive/1Fa0sm2lUglC3dyF1ctvLKO5_yneCuido?authuser=0#scrollTo=hEdFFPAyzdc0)
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

## 1. 写在前面

在深度学习领域，大模型（如 GPT-3、BERT 等）的推理是一个复杂且资源密集型的过程。这些模型的训练通常涉及大量的参数和计算资源，而推理则是在给定输入数据(prompt)的情况下生成输出或预测结果(completion)。

总的来说，大模型推理是一个两阶段的过程：

- 预填充（`prefill`）阶段：输入为全部`prompt`，输出第一个`token`，并预计算`Key-Value（KV）缓存`。关键指标是`TTFT（Time To First Token）`，即`生成第一个token所需的时间`。步骤如下：
  - ‌分词（Tokenization）‌：
    - 使用分词器（如 BPE、WordPiece）将文本转换为 Token ID 序列（例如 "Hello" → [1234]）
    - 添加特殊 Token（如 <bos>、<eos>）。
‌
  - 嵌入（Embedding）‌：
    - 将 Token ID 映射为向量：Embedding Layer 生成形状为 [seq_len, hidden_dim] 的输入

  - ‌位置编码（Positional Encoding）‌：
    - 为每个 Token 添加位置信息（如 RoPE、绝对位置编码）

  - ‌Prefill 前向计算‌：
    - 逐层计算‌：通过所有 Transformer 层处理输入序列，生成每个 Token 的隐状态
    - ‌KV Cache 生成‌：在自注意力机制中，为每个 Token 预计算并存储 Key 和 Value 矩阵（形状为 [num_layers, seq_len, hidden_dim]）
    - 输出最后一个 Token 的隐状态，作为解码的初始状态

- 解码（`decoding`）阶段：一旦`prefill`阶段完成，模型进入`decoding`阶段，逐个生成剩余的响应token。关键指标是`TPOT（Time Per Output Token）`，即`生成每个响应 token 所需的平均时间`。具体步骤如下：

  - ‌初始状态‌：
    - 输入：Prefill 阶段最后一个 Token 的隐状态
    - KV Cache：保留 Prefill 阶段的所有 Key 和 Value

  - ‌循环生成‌（每次迭代生成一个 Token）：
    - a. ‌预测下一个 Token‌：
      - 对当前隐状态进行 LM Head 投影，得到词汇表上的概率分布（logits）。
      - 根据解码策略（贪心、Beam Search、Top-k/Nucleus 采样）选择下一个 Token。
    - b. ‌更新输入‌：将新生成的 Token 作为下一步的输入（形状 ``）。
    - c. ‌更新 KV Cache‌：
      - 仅对新生成的 Token 计算新的 Key 和 Value，并追加到 KV Cache 中（显存占用随输出长度增加）。
      - 避免重新计算历史 Token 的 KV 值，使复杂度降为 O(1) 每 Token。
    - d. ‌终止判断‌：检查是否满足停止条件（如生成 <eos> 或达到 max_length）。

> ‌关键点‌：
> - Decoding 的计算复杂度为 O(1) 每 Token，但显存带宽成为瓶颈
> - 支持并行解码策略（如 Speculative Decoding）可加速生成

## 2. 大模型推理的基本步骤

### 步骤1: 输入预处理（Preprocessing）

- 文本规范化：统一大小写、去除非法字符、规范化标点等（例如将全角符号转为半角）
- 上下文拼接：若存在对话历史或多轮交互，系统会将新旧文本拼接为连续上下文（例如用特殊分隔符 <sep> 连接）
- 元指令注入：某些模型会自动添加系统提示（如："你是一个有帮助的助手..."），隐式引导生成方向

### 步骤2：Tokenization

**描述**: 将原始文本转换为模型可以理解的格式。这涉及到将单词拆分成更小的单元（通常是子词单位），以便于模型处理。

这部分操作是在 `CPU` 上完成的，因为 `tokenization` 通常不涉及大量的计算。输入是 `prompt`，经过`预训练分词器`（如 BPE、WordPiece）将文本输出为 `token ids`，输出的结果最终会被送到 GPU/NPU `显存`中已便进行下一步的计算。

例如：

- 输入：`Explain quantum computing`
- 分词结果：["Ex", "##plain", " quantum", " computing"] → [2031, 4342, 2057, 7896]

**示例代码 (Python 使用 Hugging Face 的 Transformers 库)**:

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
inputs = tokenizer("Hello, world!", return_tensors="pt")
print(inputs)
```

### 步骤3：加载预训练的大模型

**描述**: 从磁盘或其他存储位置加载已经训练好的模型权重到内存/显存中。

‌步骤‌：

- ‌加载模型文件‌：从磁盘读取包含模型权重、配置和分词器的文件（如 PyTorch 的 .bin 或 HuggingFace 格式）
- 初始化模型结构‌：根据配置初始化 Transformer 的架构（层数、注意力头数、隐藏维度等）
- 权重分配‌：将参数分配到对应的计算设备（如 GPU 显存），支持分布式推理时需分割参数到多卡
- 优化设置‌：启用量化（如 FP16/INT8）、Flash Attention 加速或显存优化策略（如 model.eval() 冻结梯度）

‌关键点‌：

- 模型加载时间与参数规模成正比（例如 7B 模型约需 15GB 显存）
- 支持动态批处理（Dynamic Batching）的框架可同时服务多个请求

**示例代码 (Python 使用 Hugging Face 的 Transformers 库)**:

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained('bert-base-uncased')
print(model)
```

### 步骤4：模型推理

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
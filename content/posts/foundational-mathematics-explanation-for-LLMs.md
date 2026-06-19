+++
date = '2026-03-11T03:41:01+08:00'
draft = false
title = '用基础数学理解 nanoGPT 的语言建模'
tags = ["LLM"]
+++

> 把语言建模问题写成条件概率：估计 $P(x_{t+1}\mid x_{\le t})$。

这篇文章更偏“看源码时能对得上号”的解释：以 nanoGPT（GPT-2 风格 Transformer）为参照（推荐对照这份带注释的实现：https://github.com/buvidk1234/nanoGPT ），只用必要的基础数学，把它为什么能做下一个 token 预测、训练时到底在优化什么，讲清楚。

约定一个更贴近实现的术语：模型直接处理的不是“单词”，而是 **token id**（通常由 BPE 等子词分词得到的整数序列）。训练后我们有时会观察到 token embedding 的几何相似性，但它不是先验语义规则，而是被目标函数与数据共同塑形的结果。

从最底层开始：如何在数值空间里表示 token，并让这种表示能够被后续计算反复使用？
## 如何让计算机表示 token？

可以借助一种数学工具——**向量**。

两个向量的夹角余弦可以衡量它们的相似度：

$$\cos(\theta) = \frac{\mathbf{a} \cdot \mathbf{b}}{\|\mathbf{a}\| \|\mathbf{b}\|}$$

$\cos(\theta)$ 越接近 1，两个向量越相似；越接近 -1，越相反。

因此，我们可以用向量来表示 token；余弦相似度常被用作“观察/分析”这种表示的一个指标（例如看哪些 token 的向量更接近）。但需要强调：

- 余弦相似度不是“语义”的定义，只是向量空间中的一个几何量。
- 向量会在训练目标（下一个 token 预测）驱动下被塑形，最终是否呈现出你直觉里的“近义/反义”取决于数据、模型容量与训练过程。

这种表示方式称为 **Token Embedding**：每个 token 对应一个 $d$ 维向量 $\mathbf{e} \in \mathbb{R}^d$。
## 如何让计算机表示序列？

同一个 token 在不同上下文里可能承担不同功能。仅靠 Token Embedding 得到的是“上下文无关”的初始表示，还需要一种方式把位置信息与上下文交互纳入计算。

为此引入**位置向量（Positional Embedding）**：假设序列长度为 $m$，为每个位置分配一个 $d$ 维向量 $\mathbf{p}_j$。在 nanoGPT / GPT-2 的常见实现里，这个位置向量通常是一个可学习的查表（learned positional embedding）。将两者相加得到模型输入：

$$\mathbf{x}_j = \text{token\_embedding}_j + \text{pos\_embedding}_j$$

直觉上理解：向量加法把“是什么 token”与“在第几个位置”两类信息叠加到同一表示上，供后续层（注意力与 MLP）继续处理。

这样，一个长度为 $m$ 的 token 序列就可以表示为 $m$ 个向量 $(\mathbf{x}_1, \mathbf{x}_2, \dots, \mathbf{x}_m)$。后续 Transformer 的计算并不是“理解句意”，而是在这个向量序列上学习一种有效的、可用于预测下一个 token 的变换。
## 如何训练模型生成文本？

GPT 类 LLM 的核心目标是**预测下一个 token**——给定前面的 token，预测最可能的下一个 token：

$$P(x_{t+1} \mid x_1, x_2, \dots, x_t)$$

例如输入“今天天气真”，模型可能给出“好”的概率较高。生成长文本时，只需不断把采样/选择出的下一个 token 追加到序列末尾，重复即可。

更贴近 nanoGPT 的训练/生成流程可以概括为：

1. 文本经分词器得到 token id 序列（整数序列）
2. token id 查表得到 token embedding，并与 positional embedding 相加
3. 经过多层 Transformer block，得到每个位置的隐藏状态 $\mathbf{h}_t$
4. 用线性层把 $\mathbf{h}_t$ 投影到词表大小的 logits：$\mathbf{z}_t \in \mathbb{R}^{|V|}$（nanoGPT 常用与 embedding 权重共享的 `lm_head`）
5. 对 logits 做 softmax 得到分布 $P(x_{t+1}\mid x_{\le t})$，训练时用交叉熵计算损失，生成时按策略采样/取最大值

其中 embedding 查表（token id → 向量）与输出投影（隐藏状态 → logits）都属于可学习参数，需要和整个模型一起通过数据学习。

**如何学习？** 用一个简单的例子来说明：

假设有一个最简单的模型 $f(x) = ax + b$，预测值为 $y$，目标值为 $y'$。我们定义一个损失函数 $L(y, y')$ 衡量误差，然后通过对参数求导得到梯度 $\nabla_a L, \nabla_b L$，据此用优化器更新参数，让损失逐步降低。

对于 LLM 这样的复杂函数，核心仍是同一件事：在大量样本上最小化“下一个 token 的负对数似然”（即交叉熵）。形式上常写成：

$$L = -\sum_{t} \log P(x_{t+1}=\text{target}\mid x_{\le t})$$

实现上就是 forward 得到 logits，计算交叉熵 loss，backward 求梯度，然后用 AdamW 之类的优化器更新参数。

## 模型的核心结构

$f(x) = ax + b$ 是线性关系，而语言建模需要更复杂的非线性函数族。Transformer 的关键并不是“引入某个神秘的数学公式”，而是用可微分的、易于并行训练的模块（注意力 + MLP + 残差 + 归一化）反复堆叠，得到足够强的函数近似能力。

以 nanoGPT（GPT-2 风格）为例，一个 Transformer block 通常长这样（省略 dropout 等细节）：

$$\mathbf{x} = \mathbf{x} + \text{Attn}(\text{LN}(\mathbf{x}))$$

$$\mathbf{x} = \mathbf{x} + \text{MLP}(\text{LN}(\mathbf{x}))$$

其中 LN 是 LayerNorm，MLP 常是两层线性层中间接 GELU 激活（也叫前馈网络 / FFN）。

### 自注意力机制（Self-Attention）

在自回归语言模型里，每个位置的新表示应当是它与可见上下文（当前位置及之前位置）交互后的结果：

$$\mathbf{x}_i^{\text{new}} = f(\mathbf{x}_i, \mathbf{x}_1, \mathbf{x}_2, \dots, \mathbf{x}_i)$$

Transformer 使用 Q、K、V 三组向量来实现这种交互。它们由同一个输入（每个位置的表示）通过三组线性变换得到——同一份输入被映射成三种“视角”：

- **Q（Query）：** $Q = \mathbf{x} W^Q$，表示“当前要从上下文中聚合哪些特征”
- **K（Key）：** $K = \mathbf{x} W^K$，表示“每个上下文位置提供的可被匹配的特征”
- **V（Value）：** $V = \mathbf{x} W^V$，表示“真正被加权汇总并输出的内容”

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V$$

其中 $\sqrt{d_k}$ 是缩放因子，避免点积幅度随维度增长而过大、使 softmax 过“尖”。$M$ 是 **causal mask**（上三角为 $-\infty$ 或很大的负数），用于遮挡未来位置，保证第 $t$ 个位置的预测只依赖 $\le t$ 的信息。这一点在 nanoGPT 里对应下三角 mask（例如 `torch.tril`）的实现。

### 多头注意力机制（Multi-Head Attention）

序列里的依存关系是多样的：局部搭配、长程指代、格式模板等。多头注意力允许模型在不同的子空间里并行学习多种对齐模式。

做法是把通道维拆成 $h$ 个头，每个头用各自的投影计算注意力，最后再拼接回去：

$$\text{head}_j = \text{Attention}(Q_j, K_j, V_j)$$

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \text{head}_2, \dots, \text{head}_h) \cdot W^O$$
## 训练中可能遇到的问题

实际模型由大量线性层 + 非线性激活（GPT-2/nanoGPT 常用 GELU）反复堆叠组成，并通过残差连接把信息与梯度更稳定地传递。在这条计算链中，常见的训练稳定性问题主要来自数值尺度与梯度传播：

### 数值爆炸 → 归一化（Normalization）

经过多层叠加后，不同通道的数值尺度可能漂移，导致训练不稳定。**LayerNorm** 会在每个位置上对特征做标准化（减均值、除方差再做可学习的缩放与平移），让各层更容易在“类似的数值范围”内工作，从而提升稳定性与可训练性。

### 过拟合 → Dropout（正则化）

如果模型过度记忆训练语料的细节，在新样本上效果会下降（过拟合）。**Dropout** 在训练时随机将一部分激活置零，相当于对不同子网络做集成，能在一定程度上缓解过拟合。在 nanoGPT 里，attention/MLP/embedding 通常都可配置 dropout。

（另外，工程里也常用 **梯度裁剪（grad clipping）** 来防止个别 batch 导致梯度过大，从而让训练更稳。）

## 回顾

回过头看，LLM 的本质可以归结为一条线索：

**把 token 映射到向量 → 用注意力在上下文中混合信息 → 用损失函数 + 反向传播学习参数 → 做下一个 token 预测并迭代生成。**

每一步都建立在基础运算之上：向量加法、矩阵乘法、softmax、求导与优化。它们本身并不“理解语言”，但在足够的数据与训练预算下，确实能学到对很多任务有用的统计结构。

## 参考资料

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) — Transformer 原始论文，提出了自注意力机制和多头注意力的完整架构
- [nanoGPT](https://github.com/karpathy/nanoGPT) — Andrej Karpathy 用约 600 行 Python 实现的最简 GPT 训练代码，适合从代码层面理解 Transformer
- [nanoGPT（含注释版本）](https://github.com/buvidk1234/nanoGPT) — 带注释的 nanoGPT 源码，适合对照本文逐段阅读实现细节
- [Let's build GPT: from scratch, in code, spelled out](https://www.youtube.com/watch?v=kCc8FmEb1nY) — Karpathy 配套的视频讲解，逐行构建 GPT
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Jay Alammar 的图解 Transformer，用可视化方式解释注意力机制

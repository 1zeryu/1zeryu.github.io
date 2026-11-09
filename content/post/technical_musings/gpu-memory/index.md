---
title: "GPU 显存：估算、优化与调试"
description: "从显存的来源讲起，算清训练和推理各需要多少显存，再总结省显存的思路与定位 OOM 的工具"
slug: gpu-memory
date: 2026-11-08 03:00:00+0800
draft: false
image: ""
categories:
    - 技术杂谈
tags:
    - 显存
    - 性能优化
authors:
  - "Yukun Zhou"
type: post
math: true
---

## 引子

跑大模型的人，大概都见过 `CUDA Out of Memory`。了解显存的基本概念，以及训练和推理各占多少显存，才能定位和解决 OOM。

本文先介绍显存的基本概念，再估算训练和推理的显存占用，然后总结降低显存用量的几种思路，最后介绍调试显存的工具和方法。

## 显存的基本概念

内存（RAM，Random Access Memory）是 CPU 使用的存储，用来存放正在运行的程序和数据；显存（VRAM，Video Random Access Memory）则是 GPU 专用的存储，物理上通常是焊在 GPU 旁边的 HBM 颗粒，供计算单元临时读写数据（模型参数、中间结果等），可参考[《AI 时代的算力芯片》](/p/compute-chips/)。

计算核心从内存读写数据走 PCIe，PCIe 5.0 x16 的单向带宽约 64 GB/s[^pcie]，而 H100 的显存带宽是 3.35 TB/s[^h100]，相差约 50 倍。矩阵计算需要反复读写权重和中间结果，数据搬运量大，如果每次都跨 PCIe 去内存取，计算核心大部分时间都在等数据，算力会被白白浪费。设计显存的目的，就是满足 GPU 计算时对数据读写的高带宽需求。

显存的容量通常在几十到上百 GB，每 GB 成本相比普通内存高得多，而且和 GPU 封装在一起，不能像内存条那样随时扩容。这源于 HBM 的封装方式。HBM 由多层 DRAM 芯片垂直堆叠，单颗容量有限；又受硅中介层面积和封装尺寸的限制，GPU 基板上的 HBM 颗粒数量通常只有几颗[^hbm]。

## 训练和推理的显存占用

估算前先统一参数量和精度这两个前提，然后从较简单的推理场景说起，再看训练的情况。

### 参数量与精度

设模型参数量为 `P`，比如 7B 模型就是 `P = 7×10⁹`。每个参数占多少字节取决于精度，字节数乘上参数量，就是模型的权重占用：

<figure>
  <table>
    <thead>
      <tr>
        <th>精度</th>
        <th>每参数字节</th>
        <th>7B 权重占用</th>
        <th>说明</th>
      </tr>
    </thead>
    <tbody>
      <tr><td>FP32</td><td>4</td><td>28 GB</td><td>单精度浮点，训练中的“高精度副本”</td></tr>
      <tr><td>BF16 / FP16</td><td>2</td><td>14 GB</td><td>半精度浮点，实际参与矩阵计算</td></tr>
      <tr><td>FP8 / INT8</td><td>1</td><td>7 GB</td><td>8 位，推理量化常用</td></tr>
      <tr><td>INT4</td><td>0.5</td><td>3.5 GB</td><td>4 位，进一步压缩权重</td></tr>
    </tbody>
  </table>
  <figcaption>各精度下每个参数的字节数，以及 7B 模型（P = 7×10⁹）的权重占用</figcaption>
</figure>

> 其中，BF16 / FP16 是目前主流的精度，能在保持模型精度的同时，降低显存占用和计算量。FP8 的硬件门槛更高：NVIDIA 从 Hopper（H100/H200）和 Ada Lovelace（RTX 40 系列）架构起才有 FP8 计算单元，更早的 A100、V100、RTX 30 系列都不支持；AMD 则是 MI300 系列起支持[^fp8]。INT8 的支持范围广得多，NVIDIA 从 Turing 架构（2018 年的 T4、RTX 20 系列）起就内置了 INT8 计算单元，所以早期的推理量化方案基本都基于 INT8[^turing]。

同一份权重，FP32 和 INT4 的占用相差 8 倍；一张 24 GB 的卡装不下 FP32 的 7B 模型，却够跑 INT4 量化版；因此，降低精度是推理时降低显存占用的直接方法，但会牺牲一定的模型精度。**为了方便对比，下文统一使用 7B 参数量、BF16 精度来理论估算。**

### 推理的显存构成

模型推理时，显存主要由三块组成：权重、临时激活和固定开销。自回归生成的模型还会多出一块 KV cache，这也是语言模型比普通视觉模型更吃显存的原因。按“是否随输入变化”，可以把推理显存拆成静态和动态两部分：

```text
推理显存
├─ 静态（加载后基本不变）
│   ├─ 权重                     14 GB（7B BF16）
│   └─ 固定开销
│       ├─ CUDA context         ~0.5 GB / 进程
│       ├─ cuBLAS/cuDNN workspace  几十–几百 MB
│       └─ 通信缓冲             ~几百 MB
└─ 动态（随请求变化）
    ├─ 激活                     ~1 GB（随 batch、分辨率、序列长度变化）
    └─ KV cache                 ~2 GB（随上下文和并发增长，仅自回归模型）
```

权重前面已经介绍过，对于 7B BF16 模型，权重占用 14 GB。这是显存中的静态占用，不会因为输入而变化。同参数量、同精度的模型，推理显存也可能差很多，差异主要来自激活值。激活值是前向计算临时产生的中间结果，会随输入变化。一般来说，视觉模型跟分辨率走，视频模型还要乘上帧数，扩散模型要为每个去噪步保留中间特征。

正常来说，激活值在推理时的生命周期较短。因为每个算子算完，大部分中间结果就可以释放，只有当前算子需要的那部分会短暂驻留，所以推理的激活通常只有几百 MB 到几 GB，远小于权重；训练则要把整个计算图的激活留到反向传播，这也是训练比推理吃显存的主要原因。

激活值没有像权重那样简单的通用公式，工程上只能按算子逐个估算：写出每个算子输出张量的形状，乘上精度字节数，沿计算图累加，取峰值作为激活的上限。这个值通常会偏大，因为很多中间结果可以复用或被算子融合掉；也会漏掉框架的 workspace 等开销。所以它只适合判断量级，精确值要靠 profiler 或显存快照实测。

固定开销种类多，下面列举几项：

| 项目 | 典型大小 | 主要作用 | 什么时候要注意 |
|---|---|---|---|
| CUDA context | 几百 MB–1 GB | 保存设备句柄、已加载的 kernel、分配器元数据 | 多进程时按进程数累加 |
| cuBLAS/cuDNN workspace | 几十–几百 MB | 矩阵乘、卷积执行时的临时草稿区 | 大 shape、开启算法 benchmark 时 |
| 通信缓冲 | 几百 MB | all-reduce、all-gather 的收发缓冲 | 多卡训练、张量并行推理 |
| 分配器缓存与碎片 | 视分配历史而定 | 复用已释放的块，避免频繁向驱动申请 | `nvidia-smi` 明显高于实际占用时 |

把这几部分放在一起看：权重和固定开销基本是定值，推理前就能算出来；激活随输入变化，但生命周期短，通常只占几百 MB 到几 GB；真正会随请求持续膨胀的，只剩 KV cache。

#### KV cache

对自回归生成的模型来说，权重之外的大头是 KV cache。自回归生成时，每生成一个 token，都要用它与前面所有 token 的 K、V 计算注意力。这些 K、V 一旦算出来就不再变化，于是缓存起来，用于后续 token 的计算；这样就能避免每次都重复计算之前的 K、V。下面根据传统的注意力计算公式，估算一下 KV cache 的大小：

$$
M_{\text{kv}} = 2 \times L \times H_{\text{kv}} \times d_{\text{head}} \times S \times B \times \text{bytes}
$$

- `2`：K 和 V 各一份；
- `L`：层数；
- `H_kv`：KV 头数，GQA/MQA 会小于总头数；
- `d_head`：每个头的维度；
- `S`：上下文长度，也就是已缓存的 token 数；
- `B`：并发序列数，也叫 batch size。

{{< details "MHA、GQA、MQA 对 KV cache 的影响" >}}

标准多头注意力（MHA）里，每个注意力头都保留自己的一组 K、V；分组查询注意力（GQA）让多组 Q 共享一组 K、V；多查询注意力（MQA）则让所有 Q 共享一组。KV cache 的大小只跟 K、V 的头数有关，所以 GQA/MQA 能按比例缩小它。

Llama-3-70B 有 64 个注意力头，但只有 8 个 KV 头，KV cache 只有标准 MHA 的 1/8。

{{< /details >}}

以 Llama-2-7B 为例（`L=32`、`H_kv=32`、`d_head=128`、BF16）：

```text
每个 token：2 × 32 × 32 × 128 × 2 = 524,288 字节 = 0.5 MiB
4K 上下文：0.5 MiB × 4096 = 2 GiB / 条序列
32 条并发：2 GiB × 32 = 64 GiB
```

权重只有 14 GB，32 条并发、4K 上下文的 KV cache 却要 64 GB，超过权重的四倍。并发越高、上下文越长，KV cache 占比越大。

{{< details "PagedAttention: KV cache 的分页管理" >}}

上面的 64 GB 是按每条序列都占满 4K 算的。真实请求长短不一，服务又没法预知每条请求最终会生成多长，只能按最大长度预留显存，大量空间实际用不上。vLLM 提出的 PagedAttention 就是为了解决这个浪费[^vllm]。

传统做法是给每条序列预留一块足够大的连续显存，比如按 8K token 算。请求结束时才释放，中间不能挪作他用；长度不同的序列反复申请、释放，还会留下用不上的空隙。论文测量，已有系统有 60%–80% 的 KV cache 显存被这样浪费，能同时服务的请求数因此大打折扣。

PagedAttention 借用了操作系统虚拟内存分页的思路：把 KV cache 切成固定大小的块（比如每块存 16 个 token），物理上不要求连续，再用一张 block table 记录每个请求的逻辑块对应哪些物理块。新 token 来了就分配新块，用完立即归还，显存按需增长；相同前缀的请求还能共享物理块，写入时再复制。这样浪费被压到 4% 以下，同样的显存能容纳更多并发请求——vLLM 的报告是吞吐提升 2–4 倍，而延迟不升。

{{< /details >}}


### 训练的显存构成

训练比推理多出梯度、优化器状态和大量激活值，这是两者显存差距的根源。下面同样用 `P` 表示参数量（7B 模型即 `P = 7×10⁹`），同样按用途拆开：

```text
训练显存
├─ 模型状态
│   ├─ 参数 + 梯度        8P（混合精度和纯 FP32 算下来一样）
│   └─ 优化器状态         取决于优化器：SGD 为 0，Adam 为 8P
├─ 激活值（跟 batch、序列长度、模型结构走）
└─ 固定开销（与推理相同）
```

#### 模型状态

模型状态由参数、梯度和优化器状态组成。

前两项的占用取决于训练精度，从最简单的 FP32 说起：参数 4 字节、梯度 4 字节，合计每个参数 8 字节。换成 FP16/BF16 混合精度后，参数和梯度各减半到 2 字节，但要额外保留一份 FP32 主权重 4 字节。这是因为 BF16 只有 7 位尾数，当更新量远小于参数本身时，BF16 的精度不足以表示更新量，如果直接在 BF16/FP16 上累加，很小的更新量可能被舍入归零；所以要在 FP32 副本上完成累加[^amp]。省下的 4 字节正好被主权重抵消，合计仍是 8 字节。FP8 训练同样是混合精度：计算时转成 FP8 只省带宽和算力，主权重和梯度一般仍按 BF16/FP32 保存，模型状态并不会按 8 倍缩小。

不同优化器要维护的额外变量数量不同，同样是出于精度考虑，这些状态默认用 FP32 保存：

| 优化器 | 每个参数的状态 | 模型状态合计 | 7B 模型 |
|---|---|---|---|
| SGD（无动量） | 无 | 8P | 56 GB |
| SGD + 动量 | 一份动量，4 字节 | 12P | 84 GB |
| Lion | 一份动量，4 字节 | 12P | 84 GB |
| Adam | 一阶、二阶动量各 4 字节 | 16P | 112 GB |
| 8-bit Adam | 两份 1 字节状态 | 10P | 70 GB |
| Adafactor | 二阶矩做因子分解，远小于 4 字节 | 约 8P + 少量 | 约 56 GB 以上 |

平时说的“模型状态 16P”，指的是混合精度 + Adam 这个最常用的组合。换优化器时，把表里对应那一行代进去即可：比如用 SGD，7B 模型只要 56 GB，一张 80 GB 的卡放得下；用 Adam 则要 112 GB，单卡必然放不下。

#### 激活值

反向传播要用到前向的中间结果，这些结果必须留在显存里。对标准的 Transformer，Megatron-LM 给出的估算公式是[^megatron-mem]：

$$
M_{\text{act}} \approx L \times \left(34 \cdot s \cdot b \cdot h + 5 \cdot a \cdot s^2 \cdot b\right) \text{ bytes}
$$

其中 `L` 是层数，`s` 是序列长度，`b` 是 micro-batch 大小，`h` 是隐藏维度，`a` 是注意力头数。公式假设 FP16、不做激活重算，并且注意力矩阵会被真实写回显存。

第一项 `34sbh` 随序列长度线性增长，来自 QKV 投影、MLP 中间层、LayerNorm 等；第二项 `5as²b` 来自注意力分数矩阵，随序列长度**平方**增长，是长上下文训练最吃显存的部分。

两项都随序列长度增长，其中第二项是平方关系。这里的序列长度不只指文本 token。对图像大模型，`s` 是图像 token/patch 的数量，随分辨率平方增长；对视频模型，则还要再乘上帧数。这就是高分辨率图像和长视频训练特别吃显存的原因。

代入 Llama-2-7B（`L=32`、`h=4096`、`a=32`），取 `b=1`、`s=2048`：

```text
线性项：34 × 2048 × 1 × 4096 ≈ 272 MiB / 层  → 32 层约 8.5 GiB
二次项：5 × 32 × 2048² × 1   ≈ 640 MiB / 层  → 32 层约 20 GiB
合计约 28.5 GiB
```

#### 其他开销

和推理一样，训练还要占 CUDA context、cuBLAS/cuDNN workspace 和通信缓冲。多卡训练的 NCCL 缓冲区更大，通常按 1–2 GB 预留。

综合起来，按 7B 模型、BF16 精度、Adam 优化器，`b=1`、`s=2048` 估算，显存占用如下：

| 项目 | 大小 |
|---|---|
| 模型状态（Adam，16P） | 112 GB |
| 激活值 | ~28.5 GB |
| CUDA context、workspace、通信缓冲 | ~1–2 GB |
| **合计** | **~142 GB** |

## 显存优化思路

在训练或推理时，经常会碰到显存不够的问题；结合前面的理论知识，可以把显存优化方法归成四类，每一类都有各自的代价：

### 降低数值精度

不改变模型结构，也不减少张量数量，只把每个数值的位宽降下来。这是代价最小、收益最直接的一类，但要考虑精度和速度上的损失。

- **混合精度训练**：参数和梯度用 BF16/FP16 存储，同时保留 FP32 主权重和 FP32 优化器状态。模型状态其实没有减少——参数省下的字节正好被主权重抵消，仍是 8 字节/参数；真正减少的是激活值：前向中间结果按 2 字节存储，激活直接减半，同时矩阵计算用低精度，带宽和算力开销也更低。
- **权重量化**：推理时把权重压到 FP8、INT8 或 INT4，每个参数从 2 字节降到 1 字节或 0.5 字节。7B 模型用 INT4 只需 3.5 GB。
- **8-bit Adam**：只改优化器状态的精度，把一阶、二阶动量从 FP32 量化到 8 位，配合块级缩放因子控制误差，优化器状态从 8 字节/参数降到 2 字节；参数、梯度和主权重仍保持原精度。代价是量化误差带来轻微的精度损失，每次更新还要做量化、反量化，速度会略慢[^8bit]。

### 减少激活值占用

激活值是前向产生、反向还要用的中间结果。它不随参数量增长，而随 batch、序列长度和输入分辨率增长，所以经常是“权重放得下，batch 或序列长度再加就 OOM”的原因。

- **激活重算（activation checkpointing）**：前向只保存每层的输入，反向需要中间结果时，用输入重新执行一遍前向。对应前面 `M_act` 的公式，每层不再保留 `34sbh + 5as²b`，只剩输入本身的 `2sbh` 字节，线性项的系数从 34 缩到 2，二次项 `5as²b` 整个不存：`2 × 2048 × 4096 × 32 ≈ 0.5 GiB`。代价是反向时重跑一次前向，约多 30% 计算。注意 0.5 GiB 是常驻部分，重算某一层时这一层的中间结果还要临时建出来，峰值约 1.4 GiB[^checkpoint]。
- **FlashAttention**：只去掉二次项 `5as²b`，把注意力计算切块，全部在片上完成，不把 `s×s` 的分数矩阵写回显存。线性项 `34sbh` 仍要保留，所以激活从 28.5 GiB 降到约 8.5 GiB[^flash]。代价是反向时没有现成的分数矩阵可用，要重新算一遍注意力；同时它依赖专用 kernel，要求 Ampere 及之后的架构、FP16/BF16 精度，对 mask 类型和 head 维度也有限制，老卡或特殊结构可能用不了。
- **micro-batch + 梯度累积**：公式里的 `b` 变小，两项都按比例下降。一次只算更小的 batch，用多次前向、反向累积梯度，效果等价于大 batch，但激活只按小 batch 计。代价是只省激活、不省模型状态；单次矩阵乘变小，GPU 利用率下降，kernel 启动和调度开销的占比上升；总计算量虽然不变，但一步训练要跑更多次前向和反向，耗时更长。

### 显存卸载（Offload）

显存装不下、但主机内存或硬盘还有余量时，把暂时不用的数据搬到显存外面，需要时再搬回来。数据总量不变，只改变存放位置，本质是用带宽换容量。

- **CPU offload**：把优化器状态、参数或激活放到主机内存，计算需要时经 PCIe 搬回，ZeRO-Offload 是典型实现。
- **NVMe offload**：把数据放到 SSD，容量更大、速度更慢，一般只在显存和内存都不够时使用。

### 并行切分

单卡容量是硬上限，实在装不下时只能把模型状态或计算拆到多张卡上，每张卡保存一部分。这是唯一能突破单卡容量限制的方法，代价是需要进行卡间通信。

<figure>
  <table>
    <thead>
      <tr>
        <th>策略</th>
        <th>切什么</th>
        <th>特点与适用场景</th>
      </tr>
    </thead>
    <tbody>
      <tr><td>数据并行（DP/DDP）</td><td>切数据：每张卡拿一份不同的 batch，用完整模型各算一遍</td><td>实现最简单，不省显存，增大了 batch size；是其他策略的基础</td></tr>
      <tr><td>ZeRO-1</td><td>切优化器状态：Adam 的一阶、二阶动量和 FP32 主权重按参数分片，每卡只留 <code>1/N</code></td><td>通信量与 DP 同量级（多一次参数 all-gather），省显存最划算的一级，模型状态吃紧时首选</td></tr>
      <tr><td>ZeRO-2</td><td>切优化器状态和梯度：反向时梯度直接 reduce-scatter 给负责对应参数的卡，不再保留完整梯度</td><td>比 ZeRO-1 再省一份梯度，通信量接近 ZeRO-1；梯度占大头时用</td></tr>
      <tr><td>ZeRO-3 / FSDP</td><td>切参数：每卡只存 <code>1/N</code> 的参数，前向、反向用到哪层就临时 all-gather 哪层</td><td>单卡模型状态能压到 <code>16P/N</code>；通信最多，单卡装不下模型时才用</td></tr>
      <tr><td>张量并行（TP）</td><td>切层内矩阵：QKV 投影和 MLP 第一层按列切，输出投影和 MLP 第二层按行切，每卡只算一部分注意力头或中间维度</td><td>单层内部就能并行，省参数也省激活；通信延迟低但频率高，必须放在 NVLink 域内，通常不超过 8 卡</td></tr>
      <tr><td>序列并行（SP）</td><td>切 TP 之后仍然重复的逐元素激活：LayerNorm、Dropout、残差沿序列维度分片，每卡只拿自己那段 token</td><td>不改变计算，只把每卡的激活再降一个数量级；和 TP 绑定使用，长序列训练时收益明显</td></tr>
      <tr><td>流水线并行（PP）</td><td>切层：按顺序把若干层放到一张卡上，micro-batch 在各段之间流水执行</td><td>通信只发生在段边界，量最小；适合跨节点，代价是流水空泡</td></tr>
    </tbody>
  </table>
  <figcaption><strong>并行策略汇总表，其中 N 为卡数。all-gather 是指把所有卡的张量拼接起来，reduce-scatter 是指把张量平均分给各卡，all-reduce 是指把所有卡的张量相加。</strong></figcaption>
</figure>

{{< details "附加补充：TP 的行切、列切与 SP 的序列切分" >}}

TP 说的“按行切”“按列切”，切的是权重矩阵的方向。以 MLP 的 `Y = GELU(X·W1)·W2` 为例，`X` 是 `s×h`，`W1` 是 `h×4h`，`W2` 是 `4h×h`，以两张卡为例：

```text
按列切 W1（h × 4h），两张卡各拿一半的列
            ┌────────┬────────┐
            │  W1_a  │  W1_b  │
            └────────┴────────┘
              GPU0     GPU1
   X 完整复制在两张卡上，各自算 X·W1_a 和 X·W1_b
   → 得到 4h 维中间结果的两半，输出维度被分开，直接拼接即可，不用通信
```

```text
按行切 W2（4h × h），两张卡各拿一半的行
            ┌─────────────────┐
            │      W2_a       │  GPU0
            ├─────────────────┤
            │      W2_b       │  GPU1
            └─────────────────┘
   每张卡用自己的那半中间结果去乘 W2_a / W2_b
   → 各得到 s×h 的部分和，必须 all-reduce 相加才是完整的 Y
```

区别在于切开的是哪个维度：切输出维度（列），各卡算的是输出的不同部分，互不重叠，拼起来就行；切输入维度（行），各卡算的是同一份输出的部分和，必须相加。所以一个 MLP 用“先列后行”的组合：`W1` 按列切，中间那个 `4h` 维度的激活自然按卡分开；`W2` 按行切，最后只做一次 all-reduce 就得到完整输出。注意力层同理，QKV 投影按列切（把注意力头分给各卡），输出投影按行切。

SP 解决的是 TP 留下的重复。TP 只切矩阵乘，`X` 本身在每张卡上仍是一份完整副本，于是 LayerNorm、Dropout、残差这些逐元素操作会对同一份数据重复计算、重复保存。SP 把 `X` 沿序列维度切开：

```text
TP：  X（s × h）→ GPU0 与 GPU1 各存一份完整的 s 个 token
SP：  X（s × h）→ GPU0 存前 s/2 个 token，GPU1 存后 s/2 个 token
```

逐元素操作不跨 token，切开后结果不变，但每张卡只存一半激活。进入注意力这种需要完整序列的计算之前，用 all-gather 把序列拼回来；反向时用 reduce-scatter 把各卡的部分梯度按序列归约回去。

{{< /details >}}

所有切分手段的代价都是通信和钱（没钱哪有这么多卡给你切分 🐶），区别在通信量和频率。DP 每步一次梯度 all-reduce；ZeRO-1/2 把它换成 reduce-scatter 加 all-gather，量相近但省显存；ZeRO-3 每次前向、反向都要 all-gather 参数[^zero]，通信最多，只能靠计算和通信重叠来掩盖；TP 每层都要 all-reduce，频率最高，所以限在 NVLink 域内；PP 只在段边界传激活，量最小，代价是空泡。


## 显存调试

前面的内容大部分是理论，算出来的都是理论值。面对复杂的实际情况，还得靠工程工具来调试，才能知道真正的瓶颈在哪里。

### 显存分配机制

PyTorch 不会每次分配都向驱动要内存，而是先申请一个大的显存池，之后从池子里切块给 tensor 用。tensor 释放时，内存回到池子里，但不一定还给驱动。所以同一个进程的显存分成三层，从外到内依次变小：

```text
nvidia-smi 看到的：进程向驱动申请的全部显存
┌─────────────────────────────────────────────────────────┐
│ CUDA context、cuBLAS/cuDNN workspace、NCCL 缓冲           │  不走 PyTorch 分配器
├─────────────────────────────────────────────────────────┤
│ PyTorch reserved：显存池                                  │
│ ┌─────────────────────────────────┬───────────────────┐  │
│ │ allocated：正在被 tensor 使用的块 │ 池里尚未复用的空闲块 │  │
│ └─────────────────────────────────┴───────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

最外层是 `nvidia-smi` 的值，即进程从驱动拿走的全部显存；去掉 context、workspace、通信缓冲这些不走 PyTorch 分配器的部分，剩下的是 `reserved`；`reserved` 再减去池里还没复用的空闲块，才是 `allocated`。所以恒有 `nvidia-smi ≥ reserved ≥ allocated`。

刚释放完一批 tensor 时，`nvidia-smi` 不会立刻下降，因为块只是回到了池子里，没有还给驱动，这属于缓存复用，并非泄漏。

PyTorch 中常用的几个接口[^torch-mem]：

- `torch.cuda.memory_allocated()`：当前还活着的 tensor 占用，对应上面的 allocated；
- `torch.cuda.memory_reserved()`：PyTorch 显存池的总大小，对应 reserved；
- `torch.cuda.max_memory_allocated()` / `max_memory_reserved()`：从进程启动到现在的峰值，比当前值更有用，OOM 往往发生在某一瞬间，而不是最后一步；
- `torch.cuda.memory_summary()`：打印按分配器、按大小分桶的明细。

#### 碎片与 expandable_segments

PyTorch 默认的分配器按固定大小的段管理显存，反复分配、释放不同大小的 tensor 后，容易留下用不上的空隙。开启可扩展段可以减少这类碎片[^cuda-mem]：

```bash
PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True python train.py
```

它不会减少显存需求，只是让同样的显存更耐用。另一个常见操作 `torch.cuda.empty_cache()` 会把池子里空闲的块还给驱动，但频繁调用会破坏复用、拖慢训练，只适合在明确知道要释放时用。

### 工具

由于篇幅原因，这里不介绍每个工具的具体用法，只列出常用的几个。个人比较常用的是 `nvidia-smi`、`torch.profiler`、memory_viz 和 Nsight Systems。

| 工具 | 调试内容 | 典型场景 | 教程 |
|---|---|---|---|
| `nvidia-smi` | 整卡显存、GPU 利用率、进程占用 | 确认是不是真占满[^smi] | [官方文档](https://docs.nvidia.com/deploy/nvidia-smi/index.html) |
| `nvtop` | 多卡显存和利用率的实时曲线 | 交互式观察趋势[^nvtop] | [README](https://github.com/Syllo/nvtop#usage) |
| `torch.cuda` | 进程内 allocated / reserved / 峰值 | 定位是哪一步涨的[^torch-mem] | [显存文档](https://pytorch.org/docs/stable/torch_cuda_memory.html) |
| memory snapshot + memory_viz | 每块显存的分配时间、大小和 Python 调用栈 | 找大块分配和泄漏[^memviz] | [实操博客](https://pytorch.org/blog/understanding-gpu-memory-1/) |
| `torch.profiler` | 算子级的时间与显存 | 找热点算子[^profiler] | [官方 recipe](https://pytorch.org/tutorials/recipes/recipes/profiler_recipe.html) |
| Nsight Systems | CPU/GPU 时间线、显存曲线 | 端到端性能分析[^nsys] | [入门指南](https://developer.nvidia.com/nsight-systems/get-started) |
| Nsight Compute | 单个 kernel 的带宽和占用率 | 优化 kernel，不解决容量问题[^ncu] | [官方文档](https://docs.nvidia.com/nsight-compute/NsightCompute/index.html) |
| DCGM | 集群多卡的显存、ECC、XID 错误 | 线上监控[^dcgm] | [用户指南](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html) |

### 排查代码

PyTorch 可以记录每次显存分配的调用栈，导出后在浏览器里查看。这是排查 OOM 最有效的手段[^mem-blog]：

```python
import torch

# 只在排查时打开，记录本身有开销，正式训练时要关闭
torch.cuda.memory._record_memory_history(max_entries=100_000)

try:
    train_one_step()
except torch.cuda.OutOfMemoryError:
    torch.cuda.memory._dump_snapshot("oom_snapshot.pickle")
    raise
```

把生成的 `oom_snapshot.pickle` 拖到 [pytorch.org/memory_viz](https://pytorch.org/memory_viz)，会看到两个视图。**Active Memory Timeline** 横轴是时间、纵轴是显存地址空间，每块矩形是一次分配，宽度是存活时长、高度是块大小（对数缩放），鼠标悬停显示大小和调用栈，还能按参数、梯度、激活等类别着色；**Allocator State History** 画的是分配器内部 segment 和 block 的已分配、空闲状态，用来判断碎片。常见的结论是“某个中间 tensor 没被释放”或“attention 的临时矩阵比预期大得多”。官方博客里有完整截图和逐步操作[^mem-blog],[^mem-blog-2]。

## 收尾

面对与 GPU 显存有关的问题时，先按理论方法估个大概，再看 `allocated`、`reserved` 和 `nvidia-smi` 的差值，最后用快照定位到具体代码。但成因五花八门：真装不下、碎片、泄漏、多进程叠加等；每个问题的解法各不相同，只能具体分析、耐心排查。受篇幅所限，还有不少内容和工具的用法没展开，需要时翻官方文档或问 AI 就行。

[^h100]: NVIDIA. NVIDIA H100 Tensor Core GPU Datasheet. https://resources.nvidia.com/en-us-tensor-core/nvidia-tensor-core-gpu-datasheet
[^hbm]: Micron. High Bandwidth Memory (HBM). https://www.micron.com/products/memory/hbm
[^fp8]: Micikevicius P, Stosic D, Burgess N, et al. FP8 Formats for Deep Learning. arXiv:2209.05433, 2022. https://arxiv.org/abs/2209.05433
[^turing]: NVIDIA. NVIDIA Turing GPU Architecture Whitepaper. https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/technologies/turing-architecture/NVIDIA-Turing-Architecture-Whitepaper.pdf
[^pcie]: PCI-SIG. PCI Express 技术概览. https://en.wikipedia.org/wiki/PCI_Express
[^amp]: Micikevicius P, Narang S, Alben J, et al. Mixed Precision Training. arXiv:1710.03740, 2017. https://arxiv.org/abs/1710.03740
[^megatron-mem]: Korthikanti V, Casper J, Lym S, et al. Reducing Activation Recomputation in Large Transformer Models. arXiv:2205.05198, 2022. https://arxiv.org/abs/2205.05198
[^flash]: Dao T, Fu D Y, Ermon S, et al. FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness. arXiv:2205.14135, 2022. https://arxiv.org/abs/2205.14135
[^checkpoint]: PyTorch. torch.utils.checkpoint: 激活重算. https://pytorch.org/docs/stable/checkpoint.html
[^vllm]: Kwon W, Li Z, Zhuang S, et al. Efficient Memory Management for Large Language Model Serving with PagedAttention. arXiv:2309.06180, 2023. https://arxiv.org/abs/2309.06180
[^zero]: Rajbhandari S, Rasley J, Ruwase O, et al. ZeRO: Memory Optimizations Toward Training Trillion Parameter Models. arXiv:1910.02054, 2019. https://arxiv.org/abs/1910.02054
[^torch-mem]: PyTorch. torch.cuda memory management. https://pytorch.org/docs/stable/torch_cuda_memory.html
[^memviz]: PyTorch. Memory visualization tool. https://pytorch.org/memory_viz
[^mem-blog]: PyTorch. Understanding GPU Memory 1: Visualizing All Allocations over Time. https://pytorch.org/blog/understanding-gpu-memory-1/
[^mem-blog-2]: PyTorch. Understanding GPU Memory 2: Find and Fix Memory Leaks. https://pytorch.org/blog/understanding-gpu-memory-2/
[^memlab]: Stonesjtu. pytorch_memlab: 按行、按 tensor 统计显存. https://github.com/Stonesjtu/pytorch_memlab
[^smi]: NVIDIA. nvidia-smi documentation. https://docs.nvidia.com/deploy/nvidia-smi/index.html
[^nvtop]: nvtop: 多卡 GPU 实时监控工具. https://github.com/Syllo/nvtop
[^profiler]: PyTorch. torch.profiler. https://pytorch.org/docs/stable/profiler.html
[^nsys]: NVIDIA. Nsight Systems User Guide. https://docs.nvidia.com/nsight-systems/UserGuide/index.html
[^ncu]: NVIDIA. Nsight Compute. https://docs.nvidia.com/nsight-compute/
[^dcgm]: NVIDIA. DCGM User Guide. https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html
[^cuda-mem]: PyTorch. CUDA semantics: Memory management. https://pytorch.org/docs/stable/notes/cuda.html#memory-management
[^8bit]: Dettmers T, Lewis M, Shleifer S, et al. 8-bit Optimizers via Block-wise Quantization. arXiv:2110.02861, 2021. https://arxiv.org/abs/2110.02861
[^qlora]: Dettmers T, Pagnoni A, Holtzman A, et al. QLoRA: Efficient Finetuning of Quantized LLMs. arXiv:2305.14314, 2023. https://arxiv.org/abs/2305.14314

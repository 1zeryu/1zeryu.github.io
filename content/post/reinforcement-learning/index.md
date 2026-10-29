---
title: "强化学习：基础"
description: ""
slug: reinforcement-learning
date: 2026-04-24 10:00:00+0800
draft: true
image: ""
categories:
    - 人工智能
    - 中文
tags:
    - Reinforcement Learning
authors:
  - "Yukun Zhou"
math: true
---

基于预训练模型，通过 SFT（监督微调）或 RL（强化学习）进一步打磨，使其在特定任务上表现更佳，已成为大模型落地的标准流程。本文主要从最基础的强化学习场景开始，介绍 RL 是如何帮助模型获取任务经验；然后介绍 RL 在目前大模型领域，如大语言模型、视觉生成模型等的应用方式。如对强化学习完全没有基础，可以先阅读 [强化学习基础](https://deepnote.com/app/ding/-deb6cadb-4777-4f46-ba81-e57b6a7bf340)。

## 强化学习理论

<a id="rl_theory"></a>
<figure>
  <img src="rl_theory.png" alt="强化学习理论" style="width:80%;" />
  <figcaption>
    强化学习理论
  </figcaption>
</figure>

与监督学习不同，监督学习依赖于大量带标签的训练数据，模型学习的是输入到输出的映射关系；而强化学习不依赖标签数据，而是通过“行动—反馈—调整”的循环，逐步学习在不同状态下采取最优行动，以最大化长期累积奖励。这种学习方式更贴近人类和动物通过经验不断改善行为的过程，因此在复杂决策、策略优化等场景中具有独特优势。





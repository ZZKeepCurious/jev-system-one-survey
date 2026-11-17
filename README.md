# System One 决策模型（Jev）技术调研：形态、机制、实现谱系与选型

这是一份面向**工程决策**的技术调研，不是论文综述。要回答的是：这类模型到底是什么、怎么训、怎么评、什么场景该用、什么场景不该用、坑在哪、上线要花多少钱。

> 正文 14 章 + 附录 A–D，约 27 万字，52 张图，115 条参考文献，37 个开源实现的范式归类。
> 证据核验至 2026-10-26（完整核验 2026-10-08）。
> 
> 在线阅读（按章分页、带章目录与公式渲染）：https://zzkeepcurious.github.io/jev-system-one-survey/

**一句话结论**：Jev 不是一个新的生成模型类别，而是一层“决策读出层”——对调用方声明的选项集合，单次前向直接读出带类型的概率分布，不生成任何文本。技术内核是把“校准”写进目标的严格适当评分规则；冻结的通用 LLM 经读出协议即可达到同型能力。

## English abstract

A technical survey of System One / typed-decision models (Jev, released 2026-09): what the paradigm actually is, how it is trained, how it is evaluated, when it pays off, and where it breaks.

Jev is not a new generative model class but a **decision readout layer** — a single forward pass returns a typed probability distribution over caller-declared options, with no text generation. The technical core is a strictly proper scoring rule with calibration written into the objective; a frozen general-purpose LLM can reach the same capability given the right readout protocol.

Scope: 14 chapters + appendices A–D, ~270k characters, 52 figures, 115 references, 37 open-source implementations classified into five training postures (P0 zero-shot readout through P4 post-hoc calibration). Evidence verified through 2026-10-26. The document is in Chinese; this abstract and the repo description are the only English on the front page.

## 三条可以先拿走的结论

**1. “该不该上决策模型范式”和“该不该买 Jev 产品”是两个独立决策。**

前者有大量公开证据支撑，后者只是商业选型。技术上已基本可以确定：开源模型零训练即可追平，微调后可进一步超越闭源 Jev。专用架构 + RLCD 不构成护城河，“开箱即用 + 接口契约”才是厂商的真实价值所在。

**2. 成本与延迟的优势属于范式，准确率的优势不属于接口。**

同模型对照读出快 5.21 倍，输入约 $0.042/MTok、输出免费。但对 15 类 Typed Decision Models 系统的独立审计显示：typed 接口相对同 backbone 的 label-logprob 读出，没有独立的精度收益。省下的是算力，不是正确率。

**3. 校准是这类范式最核心的优势，也是最需要在本地验证的性质。**

整体 ECE 合格不等于概率可逐点消费；尾部过度自信真实存在（某 31B 模型在 87% 的题上给出 ≥0.999 的概率）。而“拿通用模型原始 logit 当概率”的典型后果是 Brier 分数很差——只有经评分规则训练的读出头，才能把 logits 变成可消费的概率。

另外两条见正文：级联是当前主流落地形态，已公开的生产场景无一单独交权给决策模型；安全面已从输入侧扩展到程序消费侧。

## 实现谱系速查

按训练姿态归入五类：**P0** 零训练读出、**P1** 监督头 / 轻量微调、**P2** 数据中心化、**P3** 评分规则 RL、**P4** 后处理校准。

| 仓库 | 范式 | 读出 / 架构 | 校准手段 |
|---|---|---|---|
| TheoLeeCJ/SemIf-OpenJev | P0 (+P4) | 冻结 Qwen3.5-4B，单 token logit 读出 | 按负载独立温度（1.23 / 2.50 / 1.71） |
| Yinsongxu/LLM2Jev | P0 | prefill-only，三后端分级打分 | 无；原生零训练读出 |
| nokia-applied-research/AnyJev | P0 (+P4，闭式头) | 截断基座 + 闭式读出头 | L1 温度缩放 / L2 闭式拟合；ECE 0.03–0.05 |
| ekzhang/openjev-sglang | P0（服务层） | prefill-only + radix 缓存加速 | — |
| jaredpalmer/kev | P1 (+P2,+P4) | Qwen3.5 + LoRA r16 + pointer 头 | 按检查点独立温度（2.35 / 2.41 / 2.19 / 1.32） |
| TianyuCodings/NanoJev | P1 | Qwen3-0.6B + 三决策头 | 校准折减，未报告 ECE |
| wfzyx/von | P1 (+P4) | ModernBERT-large + Option-Marker | 条件输入温度；密封测试 ECE 0.107 |
| NandhaKishorM/laya | P3 (+P4) | ModernBERT-large + 决策头 | 分桶温度 + 直方图分箱；ECE 0.213→0.081 |
| ZimmyGao/openjev-rlcd | P3（两阶段） | Qwen3-1.7B reasoning | 温度缩放 T=1.4 |
| appliedmachinelearning-lab/jev-benchmarking | 未训练（独立评测） | 对标 2609.37647 | — |

全部 37 个仓库的范式组合、数据轴、校准参数见 [§6.2.2](https://zzkeepcurious.github.io/jev-system-one-survey/ch06.html)。

## 目录

| 章 | 内容 |
|---|---|
| [1 摘要与核心论点](https://zzkeepcurious.github.io/jev-system-one-survey/ch01.html) | 背景、五条核心判断、快速决策 |
| [2 问题定义与基本概念](https://zzkeepcurious.github.io/jev-system-one-survey/ch02.html) | 为什么“生成”是决策的错误抽象、操作定义、术语 |
| [3 校准机制与数学原理](https://zzkeepcurious.github.io/jev-system-one-survey/ch03.html) | 严格适当评分规则、朴素方案为何失败、温度缩放 |
| [4 官方体系与公开信息的边界](https://zzkeepcurious.github.io/jev-system-one-survey/ch04.html) | 三原语到四原语的 API 形态、可外部验证的计算语义 |
| [5 架构与实现形态](https://zzkeepcurious.github.io/jev-system-one-survey/ch05.html) | 官方逆向重建、六条开源路线、架构差异与开放问题 |
| [6 训练方法与数据](https://zzkeepcurious.github.io/jev-system-one-survey/ch06.html) | P0–P4 五类训练、成本规模、37 仓库映射、数据卫生 |
| [7 评测与验证](https://zzkeepcurious.github.io/jev-system-one-survey/ch07.html) | 基准解剖、五个指标、概率公理评测、最低评估协议 |
| [8 应用与落地](https://zzkeepcurious.github.io/jev-system-one-survey/ch08.html) | 已公开场景、置信度级联、影子评估、成本模型、选型决策树 |
| [9 失败模式与安全](https://zzkeepcurious.github.io/jev-system-one-survey/ch09.html) | 上下文翻转、选项语义、训练域瓶颈、输入侧攻击、防御清单 |
| [10 共识、争议与开放问题](https://zzkeepcurious.github.io/jev-system-one-survey/ch10.html) | 四条共识、六个争议焦点、证据未闭合的研究问题 |
| [11 历史谱系](https://zzkeepcurious.github.io/jev-system-one-survey/ch11.html) | 从 Brier 1950 到 Jev 发布后的生态涌现 |
| [12 实践指南](https://zzkeepcurious.github.io/jev-system-one-survey/ch12.html) | 评估协议、阈值与成本建模、上线清单、与相邻范式辨析 |
| [13 研究前沿与发展趋势](https://zzkeepcurious.github.io/jev-system-one-survey/ch13.html) | 活跃方向、演化趋势、弱信号 |
| [14 结论与展望](https://zzkeepcurious.github.io/jev-system-one-survey/ch14.html) | 技术本质、核心洞察、谱系演化、未决问题 |
| [参考文献](https://zzkeepcurious.github.io/jev-system-one-survey/refs.html) | 115 条，按来源分层标注 |
| [附录 A–D](https://zzkeepcurious.github.io/jev-system-one-survey/appendix.html) | 数学推导、术语表、速查卡、检索策略与精读清单 |

## 证据标准

- **数据来源**：arXiv 预印本、GitHub 开源实现、官方技术文档三类载体
- **时间窗**：主检索 2026-09-15 至 10-05，扩展前沿窗至 10-17，证据核验至 10-26
- **纳入流程**：首轮去重 88 篇候选 → 摘要筛选排除 15 篇 → 73 篇进候选池（其中 41 篇为新发现）→ 11 篇进入正文精读证据池
- **来源分层**：官方 / 逆向 / 转引 / 实测 / 自报，逐条标注，强度分 A/B/C
- **未能核验处如实保留**，不做补白；厂商自报数字单独标记

## 适用与不适用

**适用**：需要判断“某类决策该不该交给模型”的工程师；要为新系统做技术选型、算成本、列风险的人；在写评估方案、想知道最低可接受协议是什么的人。

**不适用**：找论文综述式文献计量的人；找可直接跳过的结论摘要的人——这份文档的结论都附了证据边界，剥离边界后就不可用。

## 引用与许可

引用格式：

```
System One 决策模型（Jev）技术调研：形态、机制、实现谱系与选型，v1.0，2026-10。
https://zzkeepcurious.github.io/jev-system-one-survey/
```

许可：本文与全部图表采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)（完整文本见 [LICENSE](LICENSE)），可自由转载、摘编、商用，只需注明作者 **ZZKeepCurious** 并附上原文链接。若发现事实错误或引文问题，欢迎在仓库 Issues 中指出。

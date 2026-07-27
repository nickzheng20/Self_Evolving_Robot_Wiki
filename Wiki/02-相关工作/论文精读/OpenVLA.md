# OpenVLA

核心判断：OpenVLA 是本项目最现实的 VLA baseline：它开源、可微调、训练在大规模 Open X-Embodiment 数据上，并且展示了 VLA 如何在新机器人任务上适配。

## 元信息

- 标题：OpenVLA: An Open-Source Vision-Language-Action Model
- 本地 PDF：`Source/Papers/openvla-2406.09246.pdf`
- 链接：[[openvla-2406.09246.pdf|本地 PDF]]
- 角色：[[VLA路线]] 的关键开源 baseline。

## 一句话版

OpenVLA 是 7B 参数开源 VLA，基于 Llama 2、SigLIP 和 DINOv2，使用 970k 机器人 episodes 训练，并支持 LoRA/quantization 等高效适配。

## 为什么重要

如果本项目未来做实验，OpenVLA 是最自然的比较对象之一。它不是封闭工业模型，论文也提供了 fine-tuning、量化、真实机器人和 LIBERO 仿真实验细节。

本项目要证明的不是“OpenVLA 不行”，而是：在某类可归因失败上，code-evolving skill layer 是否能用更少数据、更可解释、更可回滚的方式修复。

## Section-by-section

### Abstract

摘要强调两个痛点：已有 VLA 多数封闭；适配新任务的高效 fine-tuning 方法没有被充分探索。OpenVLA 提供开源模型、代码和训练管线。

论文报告 OpenVLA 在多机器人 manipulation 上优于 RT-2-X，并且能通过参数高效微调和量化在更低计算资源上部署。

### Introduction

引言认为机器人需要类似开源 LLM 生态的 VLA 基础设施。没有开放模型，研究者很难分析数据配比、架构、训练目标和部署 trade-off。

对本项目来说，这意味着未来的实验可以不只写一个传统 stack，而是把 OpenVLA 当作 learned skill 或 baseline。

### Model

OpenVLA 的结构是：

- Vision encoder：融合 DINOv2 和 SigLIP 特征。
- Projector：把视觉特征映射到语言嵌入空间。
- LLM backbone：Llama 2 7B。
- Action tokenizer：把 7 维控制动作离散成 tokens。

论文特别强调 DINOv2 的空间特征对机器人控制可能有帮助，且 VLA 训练时 fine-tune vision encoder 很重要。

### Training

OpenVLA 使用 Open X-Embodiment 中筛选出的 970k robot episodes。训练成本很高：论文报告使用 64 张 A100 训练 14 天。推理方面，bfloat16 约需 15GB GPU memory，可在 RTX 4090 上约 6Hz 运行；量化可降低部署成本。

这提醒本项目：VLA 路线强，但训练门槛高。code repair 路线的价值之一是局部修复成本低。

### Fine-tuning and Adaptation

OpenVLA 系统研究了新机器人任务上的 fine-tuning，包括 LoRA、量化、全参微调、冻结层等策略。论文发现 OpenVLA 在多任务、多对象、语言 grounding 场景中优势更明显；Diffusion Policy 在某些单任务上也很强。

这个结论支持混合路线：不是所有 skill 都该用 VLA，也不是所有 skill 都该用传统代码。

### LIBERO Experiments

论文还在 LIBERO 上测试 OpenVLA fine-tuning。结果显示 OpenVLA 平均表现最好，但和 Octo、Diffusion Policy 的差距比真实机器人实验更小。作者推测这可能是因为 OpenVLA 预训练主要是真实机器人数据，仿真有 domain gap。

这对本项目实验设计很重要：如果只在仿真里比较，VLA 的优势可能被低估或变形；如果上真实机器人，成本又更高。

## 对本项目的启发

1. OpenVLA 是现实 baseline，也是可作为 skill implementation 的候选。
2. 机器人系统可能需要“OpenVLA skill + code wrapper + safety gate”。
3. 本项目应选择 VLA 不擅长解释和局部修复的失败类型，而不是单纯比 average success。
4. 如果用 LIBERO 做实验，要注意仿真/真实数据域差异。

## 我会追问的问题

- OpenVLA 失败时，能否通过外层 code patch 修复，而不微调模型？
- 哪些任务上 OpenVLA 适合作为低层 skill，哪些任务更适合传统 planner？
- 如果 OpenVLA 被封装成 skill，precondition 和 postcondition 应怎么定义？

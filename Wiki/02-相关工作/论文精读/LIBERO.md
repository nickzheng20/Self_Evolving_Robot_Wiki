# LIBERO

核心判断：LIBERO 是本项目做 lifelong/regression 实验的关键 benchmark，因为它把机器人学习明确拆成 declarative knowledge、procedural knowledge、forward transfer、backward transfer 和 task ordering。

## 元信息

- 标题：LIBERO: Benchmarking Knowledge Transfer for Lifelong Robot Learning
- 本地 PDF：`Source/Papers/libero-2306.03310.pdf`
- 链接：[[libero-2306.03310.pdf|本地 PDF]]
- 角色：[[Benchmark与实验平台]] 中最适合测试“修复一个 skill 是否破坏旧任务”的平台。

## 一句话版

LIBERO 是一个 lifelong robot learning benchmark，包含 130 个语言条件 manipulation tasks，用来研究机器人如何在连续任务中迁移 declarative 和 procedural knowledge。

## 为什么重要

本项目最核心的实验问题不是“单次任务能不能成功”，而是：

> 修复一个 skill 后，系统是否在 lifelong setting 中整体变强，还是破坏了旧任务？

LIBERO 的 forward transfer、negative backward transfer 和 AUC 指标，正好可以改造成 skill repair 的评价体系。

## Section-by-section

### Abstract

摘要强调机器人 lifelong learning 不只是学概念，还要学动作和行为。LIBERO 提供四个 task suites、共 130 个任务，并研究五个问题：知识迁移类型、policy architecture、lifelong algorithm、task ordering robustness、pretraining effect。

论文的几个发现很重要：sequential fine-tuning 在 forward transfer 上可能优于一些 lifelong 方法；没有单一视觉 encoder 适合所有知识迁移；朴素 supervised pretraining 甚至可能伤害后续 lifelong learning。

### Introduction

引言区分 declarative knowledge 和 procedural knowledge。对机器人来说，忘记“冰箱在哪里”是 declarative forgetting；忘记“怎么打开冰箱”是 procedural forgetting。

本项目也要做这个区分。skill repair 可能修的是 procedural bug，也可能修的是 perception/world-state bug。

### Task Suites

LIBERO 有四个 suites：

- LIBERO-Spatial：空间关系变化。
- LIBERO-Object：对象类型变化。
- LIBERO-Goal：目标/行为变化。
- LIBERO-100：混合知识，包含短任务和长任务。

对本项目来说，可以把这些 suites 用作 regression matrix。例如修复 grasp_mug 后，不只看当前 mug，还看不同物体、不同空间关系、不同目标序列。

### Algorithms and Architectures

论文比较 Experience Replay、EWC、PACKNET、sequential fine-tuning、multitask learning，以及 ResNet-RNN、ResNet-Transformer、ViT-Transformer 等架构。

这部分对本项目的启发不是照搬算法，而是学习评价方式：不能只看最终成功率，还要看 forward transfer 和 backward degradation。

### Experiments

LIBERO 报告三类指标：

- FWT：forward transfer，新任务学得是否更快更好。
- NBT：negative backward transfer，旧任务是否被破坏。
- AUC：学习过程整体表现。

论文发现 task ordering 会显著影响结果，pretraining 也不一定总是帮助。这提醒本项目：skill repair 的顺序可能会影响最终库质量。先修哪个 bug、先加入哪个测试，都可能改变后续演化。

### Conclusion

论文总结 LIBERO 是支持 LLDM 研究的平台，并指出未来需要更好的架构、更好的 forward transfer algorithm、更好的 pretraining 方法。

本项目可以把 LIBERO 改造成“program repair benchmark”：不是让模型连续学习参数，而是让 coding agent 连续修复 skill。

## 对本项目的启发

1. regression rate 应成为核心指标。
2. Skill repair 要区分 declarative 和 procedural failure。
3. 修复顺序和任务顺序可能影响整体演化。
4. Benchmark 可以设计成“有缺陷 skill library + LIBERO task stream”。
5. 不要假设预训练总是帮忙；要实测。

## 我会追问的问题

- LIBERO 中哪些任务最适合注入可诊断代码缺陷？
- FWT/NBT 如何翻译成 skill patch 的 forward/backward transfer？
- 是否可以把每个失败 trace 作为 lifelong memory，而不是只训练模型参数？

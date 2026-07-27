# RoboCasa

核心判断：RoboCasa 是本项目“厨房/家庭机器人”实验场景的最佳候选之一，因为它提供大规模厨房仿真、日常任务、生成式资产和真实机器人迁移实验。

## 元信息

- 标题：RoboCasa: Large-Scale Simulation of Everyday Tasks for Generalist Robots
- 本地 PDF：`Source/Papers/robocasa-2406.02523.pdf`
- 链接：[[robocasa-2406.02523.pdf|本地 PDF]]
- 角色：[[Benchmark与实验平台]] 中最贴近“清理厨房”原始设想的平台。

## 一句话版

RoboCasa 是一个面向厨房日常任务的大规模仿真框架，包含 120 个厨房场景、2500+ 物体、100 个任务和 100K+ 轨迹，用来训练和评估 generalist robot agents。

## 为什么重要

用户的原始例子就是“清理厨房”。如果要把本项目落到可执行实验，RoboCasa 比抽象 tabletop benchmark 更贴近目标场景：厨房里有柜子、抽屉、微波炉、杯子、餐具、食材和多阶段任务。

它还能让我们构造大量 failure trace，而不会一开始就在真实机器人上冒险。

## Section-by-section

### Abstract

摘要强调机器人缺少互联网规模数据，仿真是扩展环境、任务和数据的一条路线。RoboCasa 使用逼真的厨房场景、生成式 AI 资产、LLM 辅助任务设计、人类示范和自动轨迹生成，来构建大规模训练数据。

### Introduction

引言提出仿真要满足三点：物理/渲染真实、多样性、配套大数据。RoboCasa 聚焦厨房场景，提供多种 floor plan、风格、家具、家电、物体和机器人 embodiment。

这对本项目很重要：code-evolving skill 不能只在一个桌面杯子上修好，必须面对物体、场景和布局多样性。

### Simulation

RoboCasa 基于 robosuite/MuJoCo，创建 10 种厨房 floor plan、12 种风格，共 120 个厨房场景；还有 2509 个高质量 3D objects，覆盖 153 类厨房相关物体。

这使得它适合做扰动测试：同一个 skill patch 可以在不同厨房、不同材质、不同物体类别上回归。

### Activity Dataset

任务分为 25 个 atomic tasks 和 75 个 composite tasks。composite tasks 借助 LLM 生成，覆盖洗碗、整理、准备食物、清洁等厨房活动。

对本项目来说，atomic task 可用于单个 skill repair，composite task 可用于测试高层 planner 调用修复后的 skill library 是否更稳。

### Data Generation

RoboCasa 用人类 teleoperation 收集基础演示，再用 MimicGen 自动扩展到大规模轨迹。论文报告随着机器生成数据增加，模仿学习表现提升。

这对本项目有两个启发：

- 仿真可以用来生成大量 regression scenes。
- coding agent 修复失败后，可以用自动轨迹/任务生成器补充测试。

### Experiments

RoboCasa 展示合成数据对 atomic tasks 有 scaling trend；在 composite tasks 上，fine-tuning 预训练 policy 有帮助但仍然困难；真实厨房实验中，real + sim 比 only real 表现更好。

这个结论很现实：厨房任务很难，仿真有帮助但不是万能。它适合作为本项目早期到中期平台，而不是最终证明。

### Limitations

论文指出 composite tasks 仍表现低；自动生成轨迹有 jerk/collision 等问题；LLM 任务生成仍需人工筛选；数据集缺少高度灵巧、可变形物体和双臂操作。

这正好提醒本项目：第一阶段不要做“完整清理厨房”，应从可归因、可测试的 kitchen skill 开始。

## 对本项目的启发

1. RoboCasa 适合作为厨房 skill repair 的仿真平台。
2. 可以用 25 个 atomic tasks 构造初始缺陷 skill library。
3. 可以用 75 个 composite tasks 测试 planner + skill library。
4. 生成式场景和物体适合做扰动/回归测试。
5. 真实世界迁移仍需谨慎，不能只用仿真成功声称完成。

## 我会追问的问题

- 哪些 RoboCasa atomic tasks 最容易注入可诊断代码缺陷？
- RoboCasa 的 state 信息能否生成足够好的 execution trace？
- 能否把 MimicGen 生成的失败轨迹变成 repair evaluator？

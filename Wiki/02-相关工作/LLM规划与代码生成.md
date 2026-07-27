# LLM规划与代码生成

核心判断：LLM 已经能做机器人高层规划和机器人代码生成；本项目需要在这些工作之上增加长期 skill 维护、失败归因和验证部署。

## LLM 调用已有 skill

[[saycan-2204.01691.pdf|SayCan]] 的关键思想是：LLM 提供高层语义知识，但机器人实际能做什么要由 affordance/value function 约束。它适合解释本项目里的“任务大脑”。

本项目可以借鉴 SayCan 的原则：

- 不让 LLM 选择不可执行动作。
- 每个 skill 都要有可行性估计。
- 高层计划必须被 embodiment grounding 限制。

[Inner Monologue](https://arxiv.org/abs/2207.05608) 研究 LLM 如何利用环境反馈、成功检测、场景描述和人类反馈做闭环规划。它说明 LLM 不是只能一次性规划，反馈可以让高层决策更可靠。

## 程序化计划

[ProgPrompt](https://arxiv.org/abs/2209.11302) 把机器人动作、对象和示例写成程序化 prompt，让 LLM 生成可执行任务计划。这和本项目的 skill registry 很接近：LLM 看到的不是散文描述，而是结构化 API、对象、约束和示例。

这个方向给本项目的启发是：

> skill 的 interface 设计，和模型能力同样重要。

如果 skill registry 写得模糊，coding agent 会乱改；如果 schema 清晰，agent 更像在维护一个正常软件库。

## LLM 生成 policy code

[[code-as-policies-2209.07753.pdf|Code as Policies]] 证明 LLM 可以生成 robot policy code，把自然语言任务转成 Python 逻辑、几何计算、perception API 调用和控制 primitive。

这和用户原始想法里的例子高度一致：

> 找到杯柄像素 -> 读取深度 -> 映射到三维坐标 -> 计算抓取姿态 -> 调用控制程序。

[[voxposer-2307.05973.pdf|VoxPoser]] 更进一步：让 LLM/VLM 组合 3D value maps，再由 model-based planner 生成轨迹。它说明 LLM 写代码不一定要直接输出动作，也可以生成中间表示，让传统 planner 接管低层物理执行。

## Lifelong skill library

[Lifelong Robot Library Learning](https://arxiv.org/abs/2406.18746) 与本项目非常接近：它让 LLM-based agent 持续增长 robot skill library，避免固定 skill 库限制任务范围。

本项目可以在它的基础上强调：

- 不只是增长新 skill，还要修复旧 skill。
- 修复来自真实/仿真 execution failure trace。
- 每次修复必须进入 regression suite。
- skill 版本要能回滚。

## 已精读页面

- [[SayCan]]：任务大脑如何用 affordance/value function 约束 skill 选择。
- [[Wiki/02-相关工作/论文精读/Code as Policies]]：LLM 生成 robot policy code 的直接参考。
- [[VoxPoser]]：LLM/VLM 生成 3D value maps 并交给 planner。

## 本项目要补的部分

已有工作已经覆盖：

- LLM 能做高层规划。
- LLM 能调用已有 skill。
- LLM 能写机器人 policy code。
- LLM 能构造几何/价值图/中间表示。

本项目要补的是：

> 当 skill 在物理执行中失败后，系统如何定位失败原因、提出代码 patch、验证 patch、避免 regression，并把修复沉淀进可长期维护的 skill library。

这就是 [[执行反馈自调试]] 的核心。

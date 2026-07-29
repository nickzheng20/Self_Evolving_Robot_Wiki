# 自进化代码Agent与评估器

核心判断：“自进化”不能只靠 LLM 自我感觉良好；必须有位于进化循环之外的可执行 evaluator。机器人系统的难点是 evaluator 不只运行单元测试，还要处理物理世界的多模态失败、安全约束和 harness-level 回归。

## Voyager

[[voyager-2305.16291.pdf|Voyager]] 是本项目最重要的思想近邻之一。它在 Minecraft 中让 GPT-4 持续探索、写可执行代码、根据环境反馈和执行错误修复程序，并把成功行为存入 skill library。

它给本项目的启发是：

- skill 可以是可解释、可组合、可检索的代码。
- 环境反馈可以驱动程序改进。
- skill library 会复利，降低灾难性遗忘。

但差异也很关键：Minecraft 的失败成本低，真实机器人有碰撞、力控、传感器噪声、物体破损和人类安全问题。因此本项目必须比 Voyager 多一个 [[Robot CI-CD与安全门]]。

## Eureka

[Eureka](https://arxiv.org/abs/2310.12931) 让 LLM 写 reward code，再用 RL 环境评估 reward 是否能学出好技能。它证明 coding agent 可以参与机器人学习，但它修改的是 reward，而不是直接部署到真实机器人的 skill implementation。

对本项目的启发：

- coding agent 可以承担“设计可优化目标”的角色。
- evaluator 必须能给出明确分数。
- 人类反馈可以作为安全和偏好修正。

## AlphaEvolve

[[alphaevolve-2506.13131.pdf|AlphaEvolve]] 是更通用的“代码进化”范式：LLM 修改算法代码，系统用一个或多个 evaluator 给候选代码评分，然后进行迭代优化。

它给本项目的核心启发是：

> 代码自进化的中心不是 prompt，而是 evaluator。

如果把这个迁移到机器人，evaluator 应包括：

- 成功率。
- 平均尝试次数。
- 安全违规次数。
- 旧任务 regression rate。
- 执行时间。
- 人类介入时间。
- 泛化到新物体/新位置/新光照的表现。

## Self-debugging 与动态调试

[Teaching Large Language Models to Self-Debug](https://arxiv.org/abs/2304.05128) 说明 LLM 可以利用执行反馈修复代码。

[InspectCoder](https://arxiv.org/abs/2510.18327) 进一步强调动态分析：让 LLM 主动设置断点、检查中间状态、逐步定位根因。

迁移到机器人时，对应的问题是：

- 机器人系统能不能提供足够中间状态？
- coding agent 能不能主动要求额外的仿真 probe？
- failure trace 是否能定位到 perception、planning、control 或 verification 中的具体环节？

## 从 Skill 进化到 Harness 进化

[[Harness Engineering for Self-Improvement - 综述解读|Harness Engineering for Self-Improvement]] 把自进化对象从 solution code 扩展到模型周围的 harness code，包括 context、workflow、tool description、tool implementation、middleware、sub-agent 配置和 long-term memory。

其中最值得迁移的不是“让 agent 任意修改自己”，而是 propose-evaluate-accept 协议：

1. 用 verifier-grounded trace 聚类可重复 failure pattern。
2. 只向 proposer 暴露明确可编辑面、必须保留的成功行为和过去被拒绝的 edit。
3. 每个 edit 附带根因、目标组件、预期修复和可能回归。
4. 同时运行 held-in 与 held-out tests，无回归才合并。
5. verifier、tracer、模型配置、预算和权限系统保持只读。

这也引出一个重要区分：

- **Harness-updating**：agent 能否写出结构合理的修改。
- **Harness-benefit**：运行时 planner 能否在正确状态使用修改后的 skill/tool，并提高完整任务表现。

机器人实验不能只检查 patch 通过了单元测试，还要检查高层 planner 是否实际调用新 fallback，以及系统级收益是否在 held-out task 上成立。

[[Harness VLA]] 则提供了另一侧证据：固定 primitive 和冻结 VLA 仅通过 planner、memory、re-grounding 与重试也能显著提升。因此后续 code-repair 实验必须把这种 fixed-harness improvement 作为 baseline，避免把 memory 或 retry 收益误称为 skill 自进化。

## 机器人 evaluator 的特殊性

普通代码 evaluator 通常是测试用例。机器人 evaluator 更复杂：

- 单次执行有随机性。
- 仿真和真实世界有 gap。
- 失败可能有物理损伤风险。
- 成功不是二值，可能有质量和安全程度。
- 同一个 patch 可能修复 A 场景，却破坏 B 场景。

因此，本项目的 evaluator 应是分层的：

```text
unit tests
  -> simulation score
  -> perturbation robustness
  -> regression suite
  -> safety monitor
  -> limited real-world trials
```

只有这样的 evaluator，coding agent 的“自进化”才不是口号。

## 已精读页面

- [[Voyager]]：可执行代码 skill library、反馈修复和自验证。
- [[AlphaEvolve]]：evaluator 驱动的代码进化范式。
- [[Harness Engineering for Self-Improvement - 综述解读|Harness Engineering for Self-Improvement]]：context、workflow、harness code 与 optimizer code 的自改进路线及可信边界。
- [[Harness VLA]]：固定 primitive harness 如何在不改 VLA 权重或 skill 代码时提高机器人任务可靠性。

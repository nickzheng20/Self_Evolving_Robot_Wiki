# 安全验证与机器人DSL

核心判断：真实机器人里的 LLM-generated code 必须受到 DSL、编译器、形式约束和 runtime monitor 的限制；否则“自进化”会变成不可控风险。

## 为什么不能让 LLM 自由改机器人代码

LLM 写普通软件时，错误通常由测试捕获。机器人代码的错误可能导致：

- 机械臂撞到人或环境。
- 超出关节限制。
- 夹碎物体。
- 关闭安全限制。
- 在不满足 precondition 时执行危险动作。
- 修复一个任务但破坏旧任务。

因此，本项目中 coding agent 的输出应该是 patch proposal，并且最好限制在一个 robot skill DSL 或受约束 API 里。

## NRTrans 与 Robot Skill Language

[[nrtrans-2508.19074.pdf|NRTrans]] 提出 Natural-to-Robotic Language Translation 框架，用 Robot Skill Language 抽象机器人技能，并用 compiler/debugger 验证 LLM 生成的程序。

这对本项目非常直接：skill repair agent 不应随意改底层控制，而应该在 RSL/DSL 中修改：

- skill 调用顺序。
- precondition/postcondition。
- 参数范围。
- fallback 策略。
- perception query。
- planner 配置。
- 测试描述。

底层危险动作由编译器和执行器屏蔽。

## LTL 与形式化计划

[SELP](https://arxiv.org/abs/2409.19471) 使用 Linear Temporal Logic、constrained decoding 和 fine-tuning 来生成安全高效计划。

[SafePlan](https://arxiv.org/abs/2503.06892) 使用形式逻辑和 chain-of-thought reasoner 检查 prompt、计划和任务分配安全。

[LTLCodeGen](https://arxiv.org/abs/2503.07902) 把自然语言导航任务转成语法正确的 LTL，再交给规划器。

这些工作说明：高层任务计划可以先转成形式规格，再由 planner 执行。对本项目来说，形式规格可以成为 verifier 的一部分。

[[VASO]] 把这条路线推进到 reusable skill evolution：每个 skill 同时有 planner-facing text interface 和 proposition-aligned formal interface；model checker 找到 temporal counterexample 后，把它转成 textual gradient 更新 semantic contract。它说明形式验证不仅能拒绝计划，也能生成修复证据。

但 VASO 也明确留下一个可信根问题：如果把连续机器人状态映射成 atomic propositions 的 labeling function 本身有错，形式保证会失效。因此本项目不能只说“用了 model checker 就安全”，还要把 state-to-proposition mapping、evaluator 和规格版本放入只读审计域。

## 运行时安全

[Safe LLM-Controlled Robots with Formal Guarantees via Reachability Analysis](https://arxiv.org/abs/2503.03911) 说明可以用可达性分析约束 LLM 控制系统的安全轨迹。

[SAFER](https://arxiv.org/abs/2503.15707) 使用 safety agent 和 Control Barrier Functions 来减少机器人任务规划中的安全违规。

[[SAFE]] 说明可以从 VLA latent feature 学习跨任务 failure detector，并用 conformal prediction 控制检测阈值；[[RoboCritics]] 则用 motion-level expert critics 检查碰撞、速度和危险末端姿态，并把结构化反馈返回 LLM。

这些工作提醒我们：即使 skill 代码通过了静态检查和形式 contract，真实执行时仍需要独立 runtime failure/safety monitor；检测器和 critic 默认不能由 repair agent 编辑。

## 已精读页面

- [[NRTrans]]：Robot Skill Language、compiler、debugger 和 feedback-based tuning。
- [[VASO]]：形式化 skill contract、model checking 与 counterexample-guided evolution。
- [[SAFE]]：跨任务 VLA failure monitor 与 conformal calibration。
- [[RoboCritics]]：motion-level expert critics 和可读安全反馈。

## 本项目推荐的 DSL 粒度

不要一开始设计一个宏大的机器人语言。最小可行 DSL 只需要覆盖：

```yaml
skill:
  name:
  inputs:
  outputs:
  preconditions:
  postconditions:
  safety_constraints:
  allowed_apis:
  forbidden_apis:
  fallback_policy:
  tests:
```

实现代码仍可以是 Python/C++/ROS 节点，但 coding agent 看到和修改的入口应优先是这个受约束 schema。

## 安全原则

1. LLM 可以建议，不可以直接部署。
2. DSL 可以扩展，但不能绕过安全门。
3. 每个 patch 必须说明它改变了哪些 precondition/postcondition。
4. 每个真实机器人实验都要低速、低力、可急停、有日志。
5. 任何安全约束冲突时，任务失败优先于冒险执行。

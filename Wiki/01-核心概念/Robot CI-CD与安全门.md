# Robot CI-CD与安全门

核心判断：真实机器人上的 coding agent 不能像写普通脚本一样“改了就跑”；每个 skill patch 都必须进入一个机器人 CI/CD 管道。

## 为什么需要 Robot CI-CD

机器人代码的错误会作用到物理世界。软件 bug 可能只是测试失败，机器人 bug 可能造成碰撞、夹伤、损坏物体或损坏硬件。

所以本项目中的 coding agent 应被看作“提交 patch 的工程师”，而不是“拥有部署权限的控制器”。部署权属于 verifier。

## 推荐管道

```text
Patch proposal
  -> static checks
  -> contract checks
  -> unit tests
  -> simulation tests
  -> perturbation tests
  -> regression suite
  -> safety verification
  -> limited hardware test
  -> deploy / rollback
```

### 1. 静态检查

检查代码是否通过格式、类型、导入、API 签名、依赖版本和禁止调用。

典型禁止项：

- 绕过 motion planner 直接发高频关节命令。
- 关闭力/速度/碰撞限制。
- 修改不属于当前 skill 的全局安全参数。
- 在没有 human approval 的情况下部署到真实机器人。

### 2. Contract checks

每个 skill 都要有 input schema、output schema、precondition、postcondition 和 safety constraints。patch 后这些 contract 必须仍然成立。

例如 `grasp_mug` 的 postcondition 不是“执行完轨迹”，而应是“物体在 gripper 中，且力/碰撞/关节状态安全”。

### 3. 仿真测试

仿真不是最终证明，但它是最低成本的第一道物理测试。测试应覆盖：

- 标准场景。
- 已知失败场景。
- 随机扰动：位置、光照、遮挡、物体尺寸、摩擦。
- 反例：precondition 不满足时必须拒绝执行。

### 4. 回归测试

任何修复都可能破坏旧能力。因此每个 skill 都要有 regression suite。

例如修复透明杯 depth hole 后，必须确认普通杯、带柄杯、无柄杯、侧放杯的成功率没有明显下降。

### 5. 安全验证

安全验证可以来自多层：

- DSL 编译器：参考 [[nrtrans-2508.19074.pdf|NRTrans]] 的 Robot Skill Language。
- 形式逻辑：参考 [SELP](https://arxiv.org/abs/2409.19471)、[SafePlan](https://arxiv.org/abs/2503.06892)、[LTLCodeGen](https://arxiv.org/abs/2503.07902)。
- 可达性分析：参考 [Safe LLM-Controlled Robots with Formal Guarantees via Reachability Analysis](https://arxiv.org/abs/2503.03911)。
- 控制屏障函数：参考 [SAFER](https://arxiv.org/abs/2503.15707)。

### 6. 小范围真实机器人测试

真实硬件测试应默认低速、低力、有人监督、受限工作空间，并记录完整 trace。通过后才允许扩大任务范围。

## 版本化和回滚

每个 skill 版本都应该记录：

- 修改原因。
- 失败 trace id。
- patch 摘要。
- 新增测试。
- 通过/失败的 benchmark。
- 已知风险。
- 回滚版本。

这样机器人 skill library 才像真正的软件系统，而不是一堆 prompt 生成的脚本。

## 本项目的原则

能否自进化，不取决于 coding agent 有多聪明，而取决于 evaluator 是否严格。没有 evaluator 的“自改进”只是幻觉；有 evaluator 的代码进化才可能复利。

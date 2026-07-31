# Benchmark与实验平台

核心判断：这个项目的最小可行版本应先在仿真里做“故意有缺陷的 skill + execution trace + 自动修复 + 回归测试”，再进入真实机械臂。

## 候选平台

[CALVIN](https://arxiv.org/abs/2112.03227) 适合长时序、语言条件的机器人 manipulation。它可以测试任务分解和 skill composition，但家庭环境复杂度有限。

[[libero-2306.03310.pdf|LIBERO]] 面向 lifelong robot learning，包含 130 个任务。它很适合测“修复一个 skill 是否破坏旧任务”，因为 lifelong setting 天然关注 forward transfer、backward transfer 和 task order。

[[robocasa-2406.02523.pdf|RoboCasa]] 聚焦厨房环境，有大量日常任务和资产。它和用户原始例子“清理厨房”最贴近。

[RoboCasa365](https://arxiv.org/abs/2603.04356) 在 RoboCasa 基础上扩展到 365 个日常任务、2500 个厨房环境，适合 2026 年以后做更系统的泛化评测。

[BEHAVIOR-1K](https://arxiv.org/abs/2403.09227) 覆盖 1000 个日常活动和复杂物理模拟，适合未来测试更接近真实家庭任务的长时序系统。

[RoboLab](https://arxiv.org/abs/2604.09860) 强调对 task-generalist policy 做高保真仿真分析和受控扰动测试，适合构建 regression suite。

[[HELM]] 提出的 LIBERO-Recovery 在长时序任务的 subgoal boundary 注入物体位移或 gripper-state flip，并测最终 recovery success。它适合作为 controlled perturbation 模板，但本项目还需加入 implementation defects，并区分“当场恢复”与“持久 patch 后跨任务恢复”。

[robosuite](https://arxiv.org/abs/2009.12293) 更轻量，适合早期 proof-of-concept。

[[vla-eval]] 通过隔离 model inference 与 benchmark execution 统一多种 VLA/仿真协议，可用于冻结 evaluator artifact、并行运行 final-blind episodes，并减少不同代码库预处理和依赖造成的比较偏差。

## 已精读页面

- [[LIBERO]]：适合测试 lifelong、forward transfer、backward transfer 和 regression。
- [[RoboCasa]]：适合厨房日常任务、仿真扰动和真实机器人迁移。
- [[HELM]]：LIBERO-Recovery 的受控扰动与长时序恢复协议。
- [[vla-eval]]：隔离模型和 benchmark execution 的统一评测 harness。
- [[STING]]：用 mutation testing 诊断 hidden regression suite 是否充分。
- [[CI-Repair-Bench]]：在原始多阶段 CI workflow 中验证 repository-level repair。

## 推荐 MVP 场景

不要一开始做人形机器人清理整个厨房。建议从桌面机械臂和少数 skill 开始：

- `detect_object`
- `estimate_pose`
- `grasp_mug`
- `place_object`
- `open_drawer`
- `wipe_region`
- `verify_task_success`

初始版本故意加入可诊断缺陷，例如：

- 不处理透明杯 depth hole。
- 杯柄被遮挡时仍强行抓杯柄。
- 坐标系变换少一个外参。
- IK 失败后没有 fallback。
- wipe 区域覆盖计算错误。
- postcondition 只看轨迹结束，不看物体是否真的移动。

## 指标

核心指标不应只是成功率，还应包括：

- 修复成功率：agent patch 后原失败是否消失。
- 平均修复轮数：需要几次 patch/test。
- regression rate：旧任务是否被破坏。
- safety violation count：碰撞、超力、越界等。
- generalization：新物体、新位置、新光照、新遮挡。
- human intervention time：人类需要介入多久。
- trace sufficiency：失败 trace 是否足够让 agent 定位原因。

## Baseline

primary experiment 应从同一个 frozen RPent/[[Harness VLA]] artifact 出发，采用 `Δmemory × Δcode` 对照：

1. frozen harness：不做持久适应；
2. plan/memory-only：允许 task/global memory、re-grounding、重新 staging 和 retry，但代码只读；这是 primary baseline；
3. code-only：只允许受验证的 skill implementation patch；
4. plan/memory + gated code：本项目完整方法；
5. full without gate：使用相同候选 pool，检验 verifier 的因果贡献；
6. clean/human-oracle patch：上界和 failure-type adjudication。

Direct VLA、Code as Policies/ASPIRE-style task-program generation 和 learned-policy update 可以作为背景或边界对照，但不能替代最关键的 plan/memory-only comparison。完整预算、split 和指标见 [[问题定义与实验假设]]。

## Benchmark 验收本身也要测试

[[STING]] 说明弱 regression suite 会接受语义错误 patch；[[CI-Repair-Bench]] 则强调 repository-level repair 应在原始、多阶段 workflow 中重跑。因此本项目发布 benchmark 前，应：

- 用 deliberate mutants 测 hidden suite 能否杀死 plausible-but-wrong patches；
- 把 clean/oracle patch、impact seeds、gate seeds 和 final-blind seeds 分离；
- 在无 git history、无网络、不可读 hidden tests 的 fresh snapshot 中运行 repair agent；
- 把 container、RPent/model/memory revision、evaluator 和预算写入冻结 manifest。

## 最值得构造的 benchmark

这个项目真正需要的 benchmark 不是普通“能不能完成任务”，而是：

> 给定一个有缺陷的 robot skill library 和一组失败 trace，系统能否自动修复缺陷，并在 held-out 场景中保持或提升整体成功率？

这比单纯跑现成 benchmark 更贴合项目的新意。

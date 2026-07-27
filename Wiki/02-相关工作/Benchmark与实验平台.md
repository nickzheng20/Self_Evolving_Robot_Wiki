# Benchmark与实验平台

核心判断：这个项目的最小可行版本应先在仿真里做“故意有缺陷的 skill + execution trace + 自动修复 + 回归测试”，再进入真实机械臂。

## 候选平台

[CALVIN](https://arxiv.org/abs/2112.03227) 适合长时序、语言条件的机器人 manipulation。它可以测试任务分解和 skill composition，但家庭环境复杂度有限。

[[libero-2306.03310.pdf|LIBERO]] 面向 lifelong robot learning，包含 130 个任务。它很适合测“修复一个 skill 是否破坏旧任务”，因为 lifelong setting 天然关注 forward transfer、backward transfer 和 task order。

[[robocasa-2406.02523.pdf|RoboCasa]] 聚焦厨房环境，有大量日常任务和资产。它和用户原始例子“清理厨房”最贴近。

[RoboCasa365](https://arxiv.org/abs/2603.04356) 在 RoboCasa 基础上扩展到 365 个日常任务、2500 个厨房环境，适合 2026 年以后做更系统的泛化评测。

[BEHAVIOR-1K](https://arxiv.org/abs/2403.09227) 覆盖 1000 个日常活动和复杂物理模拟，适合未来测试更接近真实家庭任务的长时序系统。

[RoboLab](https://arxiv.org/abs/2604.09860) 强调对 task-generalist policy 做高保真仿真分析和受控扰动测试，适合构建 regression suite。

[robosuite](https://arxiv.org/abs/2009.12293) 更轻量，适合早期 proof-of-concept。

## 已精读页面

- [[LIBERO]]：适合测试 lifelong、forward transfer、backward transfer 和 regression。
- [[RoboCasa]]：适合厨房日常任务、仿真扰动和真实机器人迁移。

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

建议至少对比：

1. 静态 skill library：不修复。
2. LLM planner + 静态 skill：只改计划，不改 skill。
3. Code as Policies 式一次性生成：每次任务生成代码，但不维护版本库。
4. 本项目：execution-feedback skill repair + regression + safety gate。
5. 可选 learned policy：OpenVLA、Octo、Diffusion Policy 或 ACT。

## 最值得构造的 benchmark

这个项目真正需要的 benchmark 不是普通“能不能完成任务”，而是：

> 给定一个有缺陷的 robot skill library 和一组失败 trace，系统能否自动修复缺陷，并在 held-out 场景中保持或提升整体成功率？

这比单纯跑现成 benchmark 更贴合项目的新意。

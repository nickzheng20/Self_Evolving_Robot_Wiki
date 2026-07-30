# Log

## [2026-06-28] init | 初始化代码自进化机器人 wiki

- 根据 `LLM Wiki.md` 的三层模式建立 `Source/`、`Wiki/`、`index.md`、`log.md` 与 `AGENTS.md`。
- 读取 `Source/Main Idea/机器人自我进化架构.md`，将用户想法归纳为“任务大脑 + coding agent + 可验证 robot skill library”的混合架构。
- 联网收集并整理 VLA、LLM 机器人规划、代码生成、代码自进化、安全验证和 benchmark 相关论文。
- 初始化核心页面：项目概览、核心架构、执行反馈自调试、Robot CI-CD、安全验证、相关工作、可行性与实验路线。

## [2026-06-28] source | 下载核心参考论文 PDF

- 在 `Source/Papers/` 下载 10 篇核心参考论文 PDF：Code as Policies、Voyager、AlphaEvolve、SayCan、VoxPoser、OpenVLA、RT-2、NRTrans、LIBERO、RoboCasa。
- 已用文件类型检查确认下载结果均为 PDF。

## [2026-06-28] ingest | 逐篇更新核心论文到 wiki

- 新增 [[论文入库工作流]]，沉淀以后加入论文的标准流程和页面模板。
- 在 `Wiki/02-相关工作/论文精读/` 新增 10 篇精读页：[[Wiki/02-相关工作/论文精读/Code as Policies]]、[[Voyager]]、[[AlphaEvolve]]、[[SayCan]]、[[VoxPoser]]、[[RT-2]]、[[OpenVLA]]、[[NRTrans]]、[[LIBERO]]、[[RoboCasa]]。
- 更新 `index.md`、资料清单、相关工作地图和各技术路线页，使论文精读页不成为孤立页面。

## [2026-06-28] maintenance | 将核心论文链接改为本地 PDF

- 将已下载的 10 篇核心论文链接从 arXiv 网络地址改为 `Source/Papers/` 中的 Obsidian 本地 PDF 链接。
- 保留尚未下载论文的网络链接不变。

## [2026-06-28] analysis | 补充核心论文阅读优先级

- 在 [[相关工作地图]] 中新增“核心阅读优先级”，按对“代码自进化 robot skill library”的直接贡献排序。
- 将 [[Wiki/02-相关工作/论文精读/Code as Policies]]、[[Voyager]]、[[AlphaEvolve]]、[[NRTrans]]、[[SayCan]]、[[VoxPoser]]、[[LIBERO]] 和 VLA baseline 的角色区分写入 wiki。

## [2026-07-27] ingest | 加入 Harness VLA 与 Harness Engineering

- 入库 [[Harness VLA]]：将 PDF 规范命名为 `harness-vla-2607.08448.pdf`，抽取 39 页内容，整理固定 primitive interface、两类 memory、LIBERO/LIBERO-Pro、RoboCasa365 和 RoboTwin C2R 结果，并记录其 fixed-harness 与仿真边界。
- 入库 [[Harness Engineering for Self-Improvement - 综述解读|Harness Engineering for Self-Improvement]]：明确该来源是 Lilian Weng 的综述博客，整理 context、workflow、self-harness、evolutionary search、observability 和可信 verifier 边界。
- 新增 [[机器人Harness工程]]，把 planner、skill、memory、monitor、verifier 和权限系统统一为可观测运行时，同时区分受限可编辑工作区与只读安全内核。
- 更新 [[核心架构]]、[[执行反馈自调试]]、[[Robot CI-CD与安全门]]、[[VLA路线]]、[[自进化代码Agent与评估器]]、[[相关工作地图]]、[[可行性与边界]] 和 [[研究切口与实验路线]]。
- 将 Harness VLA-style fixed harness 加入实验 baseline，防止把 memory、re-grounding 或 retry 的收益误称为 skill code evolution。

## [2026-07-28] analysis | 固化从官方 RPent/Harness VLA 起步的实验路线

- 确认 Harness VLA 官方代码入口为 [RPent](https://github.com/RLinf/RPent)，记录当前可运行主路径、Linux/CUDA 前置条件、trace artifacts、memory 只读边界和许可证待确认项。
- 在 [[研究切口与实验路线]] 中加入四级起步闸门：官方 LIBERO-Pro smoke test、单解析 primitive repair、`pi0_pick` postcondition repair、等预算端到端验证。
- 固化实验冻结变量、allowlist、四组核心对照和两周 go/no-go 标准，避免把额外 retry、预算放宽或 memory 更新误算为 code repair 收益。

## [2026-07-29] analysis | 评估 Harness VLA / RPent 作为 baseline 与 codebase

- 审计 Harness VLA 项目页、RPent 公开仓库、外置 RPent-memory、Pi0.5 checkpoint、公开 issue/PR 和当前文档。
- 新增 [[Harness VLA采用决策]]：明确 RPent 是框架/codebase、Harness VLA 是其首个论文系统和 fixed-harness 配置；baseline 为 GO，RPent 长期采用在通过工程门前为 conditional GO。
- 记录关键门：仓库许可证缺失、`0.0.0` / pre-alpha、论文公开实现覆盖不完整、rolling memory 未固定 revision、success 与 truncation evaluator 需独立复核。
- 更新 [[研究切口与实验路线]] 与 [[Benchmark与实验平台]]，前置 Milestone 0，并把 RPent fixed harness 纳入正式 baseline 梯子。

## [2026-07-29] research-design | 定义 RPent 代码修复问题与实验假设

- 新增 [[问题定义与实验假设]]，将主问题收紧为“fixed Harness VLA 穷尽 plan/memory/retry 后，受验证的局部代码修改能否产生 held-out、无回归的增量收益”。
- 按最小充分干预定义 plan/memory-repairable、code-repairable、policy-limited 与 trusted-kernel/out-of-scope，避免按终局错误码或方法结果循环分类。
- 设计 `Δmemory × Δcode` 实验矩阵、ungated ablation、hidden gate/final blind split、双预算控制、normalized recovery 与按 defect family 聚类的统计分析。
- 给出 6-case LIBERO-Pro pilot 和五份实现前必须冻结的实验规格。

## [2026-07-29] source | 摘要级登记 ASPIRE / VASO 等候选并重做 Novelty 审计

- 摘要级登记 ASPIRE、VASO、DebugRepair、HELM、SAFE、STING、vla-eval、InSight、RepairAgent、RoboCritics 与 CI-Repair-Bench 等直接相关来源；这些条目尚未下载 PDF 或完成逐篇精读。
- 记录关键 novelty 压力：ASPIRE 已覆盖多模态 trace 驱动的 robot program repair 与 skill guidance 积累；VASO 已覆盖 formal verification 驱动的 semantic skill contract evolution。
- 将可辩护切口进一步收紧为：在 frozen VLA harness 中修复跨任务共享的既有 skill implementation，并通过 `Δmemory × Δcode` 对照、hidden regression 与 safety gate 验证。
- 更新 [[相关工作地图]]、[[VLA路线]]、[[LLM规划与代码生成]]、[[自进化代码Agent与评估器]]、[[安全验证与机器人DSL]]、[[执行反馈自调试]]、[[Robot CI-CD与安全门]]、[[Benchmark与实验平台]] 与 [[问题定义与实验假设]]。

## [2026-07-29] consolidation | 统一本轮研究问题与实验口径

- 以 [[问题定义与实验假设]] 作为题目、failure taxonomy、baseline、pilot、统计和 margin 的 canonical 页面，清理旧页面中 ASPIRE 前的宽泛 novelty 与旧 baseline。
- 统一第一阶段范围：4 个 code defects + 1 个 plan/memory control + 1 个 policy control 的 LIBERO-Pro pilot；外部 evaluator/truncation 聚合属于只读 `K`，只有 skill-local postcondition/monitor 属于可编辑 `W`。
- 在 [[Harness VLA采用决策]] 记录 RPent 的实际首批代码落点 `robots/libero/tools.py` / `toolkit.py`，并增加第二 planner 与第二 repair model 的冻结后 robustness subset。
- 明确当前新文献仅为摘要级待读登记；正式入库仍需下载 PDF、建立逐篇精读页并更新 index。

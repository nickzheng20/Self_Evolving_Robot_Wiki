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

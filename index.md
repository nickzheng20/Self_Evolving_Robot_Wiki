# Index

这个 wiki 研究的问题是：能否让机器人不只依赖端到端 VLA 训练，而是通过 coding agent 持续修复和进化一套可解释、可测试、可回滚的 robot skill library。

## 入口

- [[项目概览]]：一页读懂当前项目的核心想法、结构和研究定位。
- [[核心架构]]：系统分层、数据流和 skill registry 的基本设计。
- [[相关工作地图]]：把论文和资料按技术路线放到同一张地图里。
- [[论文入库工作流]]：以后新增论文时复用的读取、写入和交叉链接流程。
- [[可行性与边界]]：哪些部分值得做，哪些地方不能过度承诺。
- [[研究切口与实验路线]]：最小可行论文/项目应该怎么做。

## 核心概念

- [[核心架构]]：任务大脑、skill registry、执行器、监控器、coding agent、验证门。
- [[执行反馈自调试]]：从真实或仿真失败 trace 到代码 patch 的闭环。
- [[Robot CI-CD与安全门]]：机器人 skill 修改后的测试、验证、部署和回滚。

## 相关工作

- [[相关工作地图]]：VLA、LLM 规划、代码生成、代码自进化、安全验证、benchmark 的总览。
- [[VLA路线]]：RT-1/RT-2、Open X-Embodiment、OpenVLA、Octo、pi0、GR00T、Gemini Robotics 等。
- [[LLM规划与代码生成]]：SayCan、Inner Monologue、ProgPrompt、Code as Policies、VoxPoser、LRLL。
- [[自进化代码Agent与评估器]]：Voyager、Eureka、AlphaEvolve 等“代码 + 反馈 + evaluator”范式。
- [[安全验证与机器人DSL]]：NRTrans、SELP、SafePlan、LTLCodeGen、SAFER 等安全与形式化约束路线。
- [[Benchmark与实验平台]]：LIBERO、CALVIN、RoboCasa/RoboCasa365、BEHAVIOR-1K、RoboLab 等。

## 论文精读

- [[Wiki/02-相关工作/论文精读/Code as Policies]]：LLM 生成机器人 policy code，是 coding agent 写 skill 的直接前作。
- [[Voyager]]：可执行代码 skill library、环境反馈和自验证的 lifelong agent。
- [[AlphaEvolve]]：代码自进化必须依赖 evaluator 的强参考。
- [[SayCan]]：LLM 高层规划与 affordance/value function grounding。
- [[VoxPoser]]：LLM/VLM 生成 3D value maps，再交给 motion planner。
- [[RT-2]]：把动作作为 token 的 VLA 路线代表。
- [[OpenVLA]]：开源、可微调的 7B VLA baseline。
- [[NRTrans]]：Robot Skill Language、compiler 和 debugger 的安全验证路线。
- [[LIBERO]]：lifelong robot learning 与 regression 评估平台。
- [[RoboCasa]]：厨房日常任务大规模仿真平台。

## 来源

- [[资料与论文清单]]：本次初始化收集的论文、报告和项目资料。
- [[机器人自我进化架构]]：用户原始想法和已有深度调研草稿。
- [[LLM Wiki]]：这个 vault 的维护模式来源。

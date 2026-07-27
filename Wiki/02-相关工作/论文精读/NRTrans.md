# NRTrans

核心判断：NRTrans 是本项目安全层的重要参考：LLM 不应直接生成自由机器人代码，而应先生成受约束的 Robot Skill Language，再由 compiler/debugger 验证和反馈。

## 元信息

- 标题：An LLM-powered Natural-to-Robotic Language Translation Framework with Correctness Guarantees
- 本地 PDF：`Source/Papers/nrtrans-2508.19074.pdf`
- 链接：[[nrtrans-2508.19074.pdf|本地 PDF]]
- 角色：[[安全验证与机器人DSL]] 的核心论文。

## 一句话版

NRTrans 定义 Robot Skill Language，让 LLM 先生成高层 RSL 程序，再由 RSL compiler 检查语法和编译成机器人控制程序；如果出错，debugger 把语义化错误反馈给 LLM 迭代修正。

## 为什么重要

本项目最危险的版本是：coding agent 随便改 Python，然后直接部署到真实机器人。NRTrans 提供了一个更稳的中间层思路：

> 自然语言任务 -> RSL -> compiler verification -> robot control program

这和本项目的 [[Robot CI-CD与安全门]] 完全一致。

## Section-by-section

### Abstract

摘要指出 LLM 直接生成机器人控制程序会产生大量编程错误，尤其轻量模型更明显。NRTrans 提出 RSL、compiler 和 debugger，在部署前提供 correctness verification，并用反馈式程序微调提升生成成功率。

### Introduction

引言强调资源受限机器人上不能总依赖超大模型；同时 LLM 输出不稳定，真实机器人又需要可靠控制程序。NRTrans 的策略是降低生成难度：不让 LLM 写完整底层控制代码，而是写更短、更语义化的技能语言。

### Framework Overview

NRTrans 分四个阶段：

1. Prompt construction and RSL generation。
2. RSL compilation and validation。
3. Feedback composition and RSL fine-tuning。
4. Robot control program execution。

这个闭环很像本项目的 skill repair loop，只是 NRTrans 主要处理“任务到程序”的正确性，本项目还要处理“执行失败后的程序修复”。

### RSL Design

RSL 的关键词来自机器人能力，每个 statement 用命令式语法表示一个 robot skill，例如 forward、approach、grasp 等。设计重点是简单、短、语义直观。

对本项目来说，RSL 可以扩展成 skill schema：

```yaml
preconditions:
postconditions:
safety_constraints:
allowed_apis:
fallback_policy:
tests:
```

也就是说，不只编译任务计划，还编译 skill 修改边界。

### Compiler and Debugger

RSL compiler 负责词法/语法检查、AST 解析和代码生成。如果发现错误，debugger 会生成更适合 LLM 理解的错误消息，例如指出关键字大小写、缺少分号、参数错误。

这个设计给本项目一个很实际的启发：不要把原始 Python traceback 全塞给 coding agent。机器人 DSL/debugger 应把错误压缩成和 skill 语义相关的反馈。

### Experiments

论文对比 ProgPrompt，报告 NRTrans 平均 program generation success rate 提升 53.6%，零样本反馈式修正相较单次生成大幅提升，轻量 2B 模型也能达到较高成功率。

这说明“语言模型能力不足”可以部分由 DSL 和 compiler feedback 补偿。

### Limitations

NRTrans 当前主要验证程序正确性，还没有充分处理环境反馈、真实物理交互、多机器人协作和复杂任务。论文未来工作也提到要集成 real-world environmental feedback。

这正是本项目可以接上的地方：把 compiler feedback 和 execution feedback 统一。

## 对本项目的启发

1. 自进化机器人必须有受约束 DSL 或 API 层。
2. compiler/debugger 是 coding agent 的反馈源。
3. 错误消息要语义化，不能只给底层 traceback。
4. RSL 可以从任务计划扩展到 skill patch schema。
5. 真实机器人部署前，compiler pass 只是第一关，不等于物理安全。

## 我会追问的问题

- 本项目的最小 DSL 是任务语言、skill schema，还是 patch language？
- 哪些修改允许 coding agent 自动做，哪些必须人类批准？
- RSL compiler 如何接入仿真和 runtime monitor？
